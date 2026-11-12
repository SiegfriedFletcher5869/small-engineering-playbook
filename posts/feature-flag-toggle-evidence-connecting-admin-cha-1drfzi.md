# Feature Flag Toggle Evidence: Connecting Admin Changes to Backend Agent Measurements

Let the admin page toggle a versioned backend feature flag, then pin its decision to each agent invocation before measuring latency and token usage. The deciding constraint is signal quality: a developer-tools agent experiment needs to distinguish a behavior change from a change in which requests were observed.

TL;DR: Let an authenticated admin page update a versioned backend flag, use the backend decision for server rendering, and carry that same decision into the agent loop. Record a compact invocation summary plus enough attempt-level evidence to explain retries. Compare latency and token consumption within equivalent workloads before deciding that a toggle helped.

This experiment note proposes a measurement design, not a measured performance result. Its evaluation constraint is explicit: an experiment cannot earn a latency win by dropping difficult tasks or hiding extra model attempts.

The interface can stay small. The evidence needs more care.

## How should an admin page toggle a feature flag without scrambling measurements?

For a concrete developer-tools example, suppose an assistant explains a repository error and can optionally perform a second retrieval pass. A small admin page turns that pass on or off. The flag controls that behavior; it does not grant permission to read a repository or execute a tool. Keep those authorization checks separate.

A tempting first design reads the flag for server rendering, reads it again when the agent starts, and attaches its current value when telemetry is exported. That makes three opportunities to describe different configurations as one run. If an administrator changes the toggle between those reads, the rendered page and the measurement label can disagree. This is a design failure to test for, not an incident report.

Freeze the decision.

The proposed backend resolves the flag once when it accepts an invocation. Store the decision value and configuration revision with the invocation identifier, and pass the snapshot into every agent step. Server rendering can use a server-resolved decision to decide which control to display. If rendering and execution are separate requests, either bind execution to an authorized, server-held snapshot or render the newly accepted decision with the execution result. Do not trust a browser-supplied boolean as authority. A displayed option is a description of an allowed behavior, not permission to invoke it.

This gives the admin page a narrow job: show the current revision, request a change, and display the backend's accepted state. A Python backend can own the decision and measurements while a separate web application owns server rendering. The example below isolates that backend boundary; it is not a framework-specific page implementation.

## Keep the control path small and explicit

Use an authenticated read operation for the admin state and an authorized mutation for changes. In this proposed contract, a change includes the revision the administrator saw. The backend compares that expected revision with stored state and commits the update atomically. If they differ, reject the stale edit and return the current state for review. The reason is practical: two open admin tabs should not silently overwrite each other's decisions.

For cookie-authenticated mutations, include a CSRF defense appropriate to the application. Record the actor, accepted revision, and change time in an access-controlled audit record. Never put credentials or raw prompts into that record. Treat the flag state and admin response as private configuration rather than a publicly reusable page response; otherwise, a shared rendering cache becomes another place where the visible decision can drift.

There is a trade-off here. Reading authoritative state for every accepted invocation makes configuration freshness easier to reason about, but adds a dependency on the configuration store. A local snapshot reduces that dependency while introducing a propagation interval. Choose a freshness bound that the experiment can tolerate, record the revision actually used, and test the chosen behavior during configuration-store unavailability. An explicit default should still produce a labeled decision so those invocations remain visible in analysis.

A rollback changes future accepted invocations under this design. Existing runs retain their snapshots.

If a feature needs immediate cancellation for safety or access control, implement that as a separate check with its own evidence; quietly changing an experiment halfway through destroys the meaning of its label.

## A focused Python example: preserve the evidence across attempts

The useful notebook-to-prod boundary is the measurement wrapper. Notebook experiments and the deployed worker should agree on what constitutes an invocation, an attempt, and missing usage. Otherwise, a tidy evaluation chart can conceal different accounting rules.

Here is a standard-library example of that wrapper. Its illustrative flag enables a second retrieval pass; the injected agent implementation owns retrieval and model calls. Each model attempt must append its usage, including retry attempts. A missing usage value remains unknown rather than becoming zero. The returned dictionary is ready for a caller to serialize to its telemetry sink. No provider SDK or network endpoint is implied.

```python
from dataclasses import dataclass
from time import perf_counter
from typing import Callable


@dataclass(frozen=True)
class Decision:
    second_pass: bool
    revision: int


@dataclass(frozen=True)
class Attempt:
    input_tokens: int | None
    output_tokens: int | None

    def __post_init__(self):
        for value in (self.input_tokens, self.output_tokens):
            if value is not None and value < 0:
                raise ValueError("Token counts must be nonnegative")


def token_sum(attempts: list[Attempt], field: str) -> int | None:
    values = [getattr(attempt, field) for attempt in attempts]
    if not values or any(value is None for value in values):
        return None
    return sum(values)


def observe(
    invocation_id: str,
    decision: Decision,
    run_agent: Callable[[Decision, list[Attempt]], None],
) -> dict:
    attempts: list[Attempt] = []
    started = perf_counter()
    status = "ok"
    try:
        run_agent(decision, attempts)
    except Exception:
        # Record a bounded status; keep exception text out of telemetry.
        status = "error"
    return {
        "invocation_id": invocation_id,
        "flag_revision": decision.revision,
        "second_pass": decision.second_pass,
        "status": status,
        "duration_ms": (perf_counter() - started) * 1000,
        "attempt_count": len(attempts),
        "input_tokens": token_sum(attempts, "input_tokens"),
        "output_tokens": token_sum(attempts, "output_tokens"),
    }


def simulated_agent(decision: Decision, attempts: list[Attempt]) -> None:
    # Synthetic accounting values, not measured provider responses.
    attempts.append(Attempt(input_tokens=120, output_tokens=30))
    if decision.second_pass:
        attempts.append(Attempt(input_tokens=180, output_tokens=45))


example = observe(
    "synthetic-invocation",
    Decision(second_pass=True, revision=7),
    simulated_agent,
)
assert example["input_tokens"] == 300
assert example["output_tokens"] == 75
assert example["attempt_count"] == 2
```

Those numbers are synthetic fixtures.

They demonstrate aggregation, not a latency benchmark or an estimate of production consumption. The elapsed duration includes Python execution in this example; real model latency only enters when the injected function performs that work.

There are deliberate limits. This wrapper returns an error status instead of an agent result, so it belongs at a measurement boundary whose caller checks that status. It has no durable delivery, concurrency handling, or process-crash recovery. Production integration needs those decisions explicitly: persist the accepted invocation, emit a completion record, and reconcile invocations that never complete. Deduplicate summaries by invocation identifier before aggregation. Otherwise, an exporter retry can look like another user request.

Keep invocation identifiers in detailed records for joining evidence. Use bounded dimensions such as decision, outcome, and workload class for aggregate charts. Putting every prompt or invocation identifier into a chart dimension creates more series while adding little comparative signal.

## Compare behavior without rewarding missing data

Start with an eval harness that holds the repository task, acceptance rubric, and model configuration constant. Run both decisions against the same versioned task set, preserving the configuration revision in the result. For an agent that explains errors, the rubric might require an explanation grounded in the supplied repository context and a useful next diagnostic step. That is a proposed evaluation criterion, not a claim that token counts measure answer quality.

Count failed tasks.

A chart containing only successful invocations can make the faster-looking decision attractive precisely because its slow or difficult tasks disappeared. Report accepted invocation count, completion count, error count, and missing-usage count beside latency and token distributions. Keep successful-task latency and all-invocation completion behavior distinguishable. A timeout must remain an outcome in the denominator even when its final provider usage is unavailable.

For the synthetic example, one invocation has two attempts. Counting each attempt as an independent task would give the second-pass decision a different denominator. Summing only the last attempt would lose 120 input tokens and 30 output tokens. Both mistakes change the experiment without changing the agent. This is why the summary and attempt records need an explicit relationship.

Token usage also needs context before it becomes money. Preserve the model identity, usage categories reported by the provider, and the applicable accounting version in the production record. Convert those categories using a documented rate schedule when reporting monetary cost. A single total-token field is useful for a narrow synthetic example, but it is insufficient for a general billing calculation.

Don't fill missing usage with a guessed price.

For grouping repeated errors, choose a stable failure category such as retrieval timeout or invalid tool input. Public event-grouping documentation describes fingerprints as a way to control which events belong together; that is useful background for the general distinction between an individual event and its error family. A request identifier belongs in the event's join fields, not in a grouping key intended to collect repeated failures. Keeping these roles separate makes it possible to inspect one broken run without fragmenting the overview into one group per request.

## What should be measured before copying this design?

Before rollout, exercise a toggle change during a running invocation, a stale admin edit, a failed model attempt with unknown usage, and a duplicated telemetry delivery. These are test cases, not claimed test results. Verify the expected invariants: the accepted decision stays attached to the run, stale writes are rejected, unknown usage stays visible, and replay does not increase the invocation count.

Then measure instrumentation overhead with the same workload and execution environment, including queueing time if users experience it. Set an acceptable overhead budget before interpreting the experiment. More detailed records can explain a regression, but exporting everything synchronously can itself alter latency; moving export off the request path introduces delivery and reconciliation work. That choice deserves measurement too.

Finally, minimize retained content. Prefer task identifiers and bounded diagnostic fields to copied repository files, prompts, or generated answers. If retained records contain personal data, design a way to locate and erase it where required: Article 17 describes the right to erasure and its conditions and exceptions. This note does not establish a universal retention period or claim that every telemetry record must be deleted on request. The engineering requirement is to understand what was stored and where its copies went.

**Adopt the toggle only when task quality, latency, and usage completeness support the same decision.** A concise admin page can be enough. The durable result is an experiment whose labels survive rendering, retries, configuration changes, and export, so a notebook finding can still mean something after deployment.

## References

- Event grouping and fingerprint mechanics: https://docs.sentry.io/concepts/data-management/event-grouping/
- GDPR Article 17, right to erasure: https://gdpr-info.eu/art-17-gdpr/
