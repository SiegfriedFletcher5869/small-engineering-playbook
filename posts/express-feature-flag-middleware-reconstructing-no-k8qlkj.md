# Express Feature Flag Middleware: Reconstructing Node.js Pricing Decisions Per Request

A pricing rollout is only reconstructable if checkout retains the decision it actually used. Reading today's flag after a disputed order cannot establish yesterday's decision. TL;DR: use an Express middleware guard for each privileged request, retain its evaluated flag value with the request's pricing evidence, and connect that evidence to the preparation job. Evaluate the design against a stricter constraint: can another engineer explain an individual order without rerunning a model or guessing which configuration was active?

The design result is an evidence chain, not a measured performance result. For an e-commerce team moving a Python pricing experiment from notebook-to-prod, that chain should connect the preparation run, approved rule, flag decision, and resulting order. Keep the model's recommendation separate from the server's authority to apply it. This experiment note focuses on that boundary; the Python example exercises the evidence handoff that an Express guard would consume.

## What should Express middleware retain from each feature flag check?

Consider a proposed fixture with 3 checkouts: an eligible account, an ineligible account, and an eligible account arriving after rollback. These are test cases. The question is whether retained evidence distinguishes them. A flag name identifies a control, while the evaluated value identifies what the application saw at a particular decision point. Your application should retain the request identifier, its own targeting attributes, the selected flag key, the observed decision, and the pricing-rule version alongside the order. Those are proposed application fields, not a vendor response schema. Store enough context to explain the decision without copying an entire customer profile into an investigation record.

Capture the decision.

The simple approach checks a flag in the browser and looks up its current state during investigation. That leaves the privileged route exposed to callers who skip the UI, and it loses the historical decision. An Express middleware pattern addresses the first problem: read the configured key, evaluate its enabled state or value, and allow or block route execution before applying the pricing rule. Application-owned evidence addresses the second. Preserve the exact decision used for the order rather than evaluating again halfway through the request.

A notebook may propose a rule, and an eval harness may score it, but neither result authorizes a checkout. The server does. Store your own targeting attributes and map them to separate flag keys where rollout conditions require it; keep that mapping versioned with the application rule. Treat flag checks and account authorization as separate checks that must both succeed.

## Carry preparation evidence into the flag check

A preparation job might produce the candidate pricing rule that checkout will later use. The useful seam is between inspecting that job's evidence and permitting the next flag lookup. This Python program performs two read-only API calls using one credential and one base URL. It retrieves a specified cron run, checks its complete JSON response against an application-approved fingerprint, and only then retrieves the configured pricing flag value. The first response controls whether the second capability executes.

The approval fingerprint is an intentionally narrow experiment constraint. Matching it means that the response matches an approved snapshot; it does not infer job success from undocumented response fields. An application reviewer must establish what that snapshot means before approving it. If the response contains changing metadata, the full-response comparison will reject it. For production, define a versioned projection over documented fields and review that projection with the pricing artifact. Don't quietly discard fields until a comparison happens to pass.

Use Python 3 with the standard library. Configure INFRAI_BASE_URL to the documented API origin with its version prefix, and set INFRAI_API_KEY, PRICING_CRON_ID, PRICING_RUN_ID, PRICING_FLAG_KEY, and APPROVED_RUN_SHA256. The final variable contains the SHA-256 fingerprint of a previously reviewed response serialized with the canonicalization below. The flag must already exist. A mismatch prints its fingerprint for independent review; automatically approving that fingerprint would defeat the check.

```python
import hashlib
import json
import os
import time
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = os.environ["INFRAI_BASE_URL"].rstrip("/")
API_KEY = os.environ["INFRAI_API_KEY"]

def canonical(value):
    return json.dumps(value, sort_keys=True, separators=(",", ":"),
                      ensure_ascii=True).encode("utf-8")

def get_json(path):
    for attempt in range(4):
        request = Request(
            BASE_URL + path,
            headers={"Authorization": f"Bearer {API_KEY}",
                     "Accept": "application/json"},
            method="GET",
        )
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"Unexpected status: {response.status}")
                return json.load(response)
        except HTTPError as error:
            detail = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 3:
                raise RuntimeError(f"HTTP {error.code}: {detail}") from error
            delay = float(2 ** attempt)
            retry_after = error.headers.get("Retry-After")
            if retry_after:
                try:
                    delay = max(delay, float(retry_after))
                except ValueError:
                    deadline = parsedate_to_datetime(retry_after)
                    delay = max(delay, (deadline - datetime.now(timezone.utc))
                                .total_seconds())
            time.sleep(delay)
    raise RuntimeError("Retry budget exhausted")

cron_id = quote(os.environ["PRICING_CRON_ID"], safe="")
run_id = quote(os.environ["PRICING_RUN_ID"], safe="")
flag_key = os.environ["PRICING_FLAG_KEY"]
run_evidence = get_json(f"/cron/runs/get/{cron_id}/{run_id}")
fingerprint = hashlib.sha256(canonical(run_evidence)).hexdigest()
if fingerprint != os.environ["APPROVED_RUN_SHA256"].lower():
    raise RuntimeError(f"Run evidence needs review: {fingerprint}")

flag_evidence = get_json("/flags/get_value/" + quote(flag_key, safe=""))
print(json.dumps({
    "observed_at": datetime.now(timezone.utc).isoformat(),
    "run_sha256": fingerprint,
    "flag_key": flag_key,
    "flag_response": flag_evidence,
}, sort_keys=True))
```

Both calls share the same base URL and credential. The program preserves the flag response without guessing its envelope or coercing a JSON object into a Boolean. The application adapter must interpret the documented value before an Express guard can authorize execution. This is a runnable evidence probe; installing it as middleware still requires that adapter and a durable evidence store.

That distinction matters.

A nonempty response object is truthy in Python even when the flag value inside it is false. An eval-driven implementation should include that case before carrying notebook logic into a server. Retain the resulting evidence in the application's order or audit store; stdout alone is only probe output. Both requests are reads, so this example performs no pricing mutation that retries could apply twice.

## Compare the investigation work behind the flag

Infrai fits a team that wants jobs and observability behind one API key and one bill, reducing credential sprawl and invoice reconciliation across backend services; its public, self-describing discovery surface provides request and response schemas for building the adapter. The application still needs its own decision history and reviewed targeting mapping. A combined provider means one vendor to trust and one shared outage surface.

For a fair comparison, distinguish the jobs other products occupy in the alternative stack. Amazon SQS is the queue component in an SQS DLQ plus Sentry Crons design; Sentry Crons occupies the scheduled-job monitoring role. Healthchecks is another tool to consider for whether an expected task checked in. These are different selection boundaries: queue delivery, scheduled execution monitoring, and check-in monitoring. None of those roles, by itself, establishes which pricing decision a checkout used. Treat that as application evidence regardless of vendor. The references are evaluation entry points, not evidence of a benchmark or feature-parity test.

The SQS plus Sentry alternative requires two provider accounts or signups for a team starting from scratch, two credential sets, and application glue connecting queue messages, dead-letter context, scheduled-run identity, and captured errors. Existing accounts change the signup work, but they don't remove the correlation work. A team already operating those systems may prefer keeping them: adopting a combined provider still requires migration and an adapter. A small Python service may place greater weight on sharing credentials and response conventions.

For teams evaluating an existing Grafana workflow or a Prometheus-based metrics convention, use a different acceptance question from the Sentry Crons check-in test. The following comparison separates those questions without assuming any product stores the application's historical pricing decision.

| Candidate | Appropriate evaluation context | Boundary to verify |
| --- | --- | --- |
| Sentry Crons | Reconstructing scheduled preparation activity | Correlate the run with the order's retained flag decision. |
| Prometheus | Naming the measurements used to assess the rollout | Aggregate measurements must be accompanied by individual order evidence. |
| Grafana | Keeping investigation in a workflow the team already uses | Demonstrate navigation to the retained decision before accepting the integration. |

These are complementary candidates, so replacing one with another is not a feature-parity claim. Prefer an existing investigation workflow when it can satisfy the evidence test; a new dashboard alone is no reason to migrate backend services.

Make the acceptance test concrete: can an investigator move from the order to its preparation run and any captured error using stable application identifiers? Runs, dead-letter context, and errors belong in that investigation plan. The two-call probe demonstrates run evidence flowing into a flag lookup. It does not establish automatic correlation or historical flag auditing. Test those requirements separately before selecting a stack, and retain the order's decision even if a monitoring product also stores related events.

## Measure before copying the choice

Start with the 3 checkout fixtures and add a false flag value wrapped in a nonempty JSON response, a changed preparation snapshot, and a rate-limited read. Require the guard to block privileged execution when it cannot establish an allowed decision. An investigation should recover the decision already recorded for an order even after the live flag changes. These are proposed acceptance criteria.

Brief application caching can reduce repeated flag polling when many routes depend on the same key. It creates a freshness trade-off. Record the cache observation time with the decision, choose the permitted rollback delay explicitly, and test requests that straddle that boundary. A request served from a cache still needs its own decision record. Otherwise a later investigator may confuse the time the request arrived with the time its flag value was observed.

Keep the harness prompt-cost aware: this boundary is deterministic, so replay retained evidence and rule fixtures before spending tokens on another model evaluation. Measure evidence completeness, reconstruction time, and stale-decision exposure first. Then evaluate the pricing assistant's recommendations separately. The notebook-to-prod transition has a useful finish line: an engineer can explain why a particular checkout received the new rule without regenerating the model's answer.

## Sources

References for evaluating the alternative stack and surrounding observability conventions:

- Amazon SQS developer guide: https://docs.aws.amazon.com/AWSSimpleQueueService/latest/SQSDeveloperGuide/welcome.html
- Sentry Crons documentation: https://docs.sentry.io/product/crons/
- Healthchecks documentation: https://healthchecks.io/docs/
- Grafana documentation: https://grafana.com/docs/
- Prometheus metric naming practices: https://prometheus.io/docs/practices/naming/
- RFC 5424, log severity semantics: https://datatracker.ietf.org/doc/html/rfc5424
