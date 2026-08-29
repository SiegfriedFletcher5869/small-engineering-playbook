# Python Logistics Transcription: 3 Queue Controls for 429 Retry-After Limits

**Short answer:** Handle speech-to-text 429 responses with a bounded queue that honors Retry-After, uses jittered backoff only when no usable delay is supplied, and stops retrying at a ticket-specific deadline.

For logistics support, the operating target isn't maximum throughput. It is the best transcription quality that still gets an urgent damaged-shipment call in front of an agent on time.

Don't let each web request retry on its own. That creates a second arrival wave after the first limit response, while long audio waits beside short, urgent clips. Put admission, scheduling, and retry timing in one place; keep the speech API behind a small transport adapter; and measure queue age separately from API latency.

This is a queueing problem first.

Backpressure is the feature.

## How should a Python speech-to-text API queue handle 429 Retry-After limits?

The data flow is plain: the ticket endpoint stores the audio reference and priority, a scheduler admits work, a worker calls an injected transcription transport, and the result is attached to the ticket. A 429 returns to the scheduler with either a parsed delay or no delay. The worker does not sleep while holding a concurrency slot. It reschedules the job, releases capacity, and moves on.

The example below is deliberately vendor-neutral. `transcribe` is the only integration point, so its adapter can translate a provider response into either text or `RateLimited`. The sample transport simulates a sequence of responses; replace it with the transport already used by the application. No route shape is assumed.

```python
from __future__ import annotations

import asyncio
import heapq
import random
import time
from dataclasses import dataclass, field
from typing import Awaitable, Callable


class RateLimited(Exception):
    def __init__(self, retry_after_seconds: float | None = None) -> None:
        self.retry_after_seconds = retry_after_seconds


@dataclass(order=True)
class Job:
    ready_at: float
    priority: int
    ticket_id: str = field(compare=False)
    audio_ref: str = field(compare=False)
    deadline: float = field(compare=False)
    attempts: int = field(default=0, compare=False)


Transcribe = Callable[[str], Awaitable[str]]


def retry_delay(attempt: int, retry_after: float | None) -> float:
    if retry_after is not None and retry_after >= 0:
        return retry_after
    ceiling = min(2 ** attempt, 30)
    return random.uniform(0, ceiling)


async def run_queue(
    jobs: list[Job],
    transcribe: Transcribe,
    save_result: Callable[[str, str], None],
) -> None:
    heapq.heapify(jobs)

    while jobs:
        job = heapq.heappop(jobs)
        now = time.monotonic()
        if job.ready_at > now:
            await asyncio.sleep(job.ready_at - now)

        if time.monotonic() >= job.deadline:
            save_result(job.ticket_id, "DEFERRED_FOR_REVIEW")
            continue

        try:
            transcript = await transcribe(job.audio_ref)
            save_result(job.ticket_id, transcript)
        except RateLimited as error:
            job.attempts += 1
            delay = retry_delay(job.attempts, error.retry_after_seconds)
            job.ready_at = time.monotonic() + delay
            heapq.heappush(jobs, job)


async def demo() -> None:
    responses: list[float | str] = [2.0, "forklift arrived with a broken seal"]
    saved: dict[str, str] = {}

    async def simulated_transport(audio_ref: str) -> str:
        response = responses.pop(0)
        if isinstance(response, float):
            raise RateLimited(retry_after_seconds=response)
        return response

    now = time.monotonic()
    jobs = [
        Job(
            ready_at=now,
            priority=0,
            ticket_id="T-1842",
            audio_ref="stored-audio/T-1842.wav",
            deadline=now + 20,
        )
    ]
    await run_queue(jobs, simulated_transport, saved.__setitem__)
    print(saved)


asyncio.run(demo())
```

Three controls matter here. `ready_at` prevents an immediate retry loop. `priority` lets the application order ready jobs by business urgency. `deadline` turns an exhausted latency budget into an explicit review state instead of an infinite retry. In production, the queue must live in durable storage rather than process memory, and multiple workers need an atomic claim operation. Those are deployment properties, not reasons to tangle provider-specific logic into the scheduler.

Consider the full path for the sample damaged-seal ticket. The endpoint accepts `T-1842`, stores its audio reference, and returns without waiting for speech processing. A worker claims the job and the adapter reports a 429 with a usable 2-second delay. The scheduler writes the new eligibility time, releases the claim, and can immediately admit another ready ticket; it does not park a worker for 2 seconds. When `T-1842` becomes eligible again, the scheduler compares its priority and deadline with the other ready jobs before making another call. A successful transcript is attached once under the stable ticket job identifier. If the deadline arrives first, the state changes to `DEFERRED_FOR_REVIEW`, preserving the original audio for an agent instead of quietly switching to a lower-quality configuration. This example contains no promise that 2 seconds is universally correct — it demonstrates why the service-supplied delay, application deadline, and business priority are separate values. Keeping those values separate also makes the flow testable: a fake clock and scripted adapter can cover the transition without sending audio or waiting in real time.

The short simulated case uses a 2-second delay and ticket `T-1842` so the state transition can be inspected. It isn't a benchmark or a recommended service limit. The application should record the raw rate-limit outcome in structured telemetry, but it should expose only the normalized delay and outcome to the scheduling layer. This separation keeps notebook experiments honest: the same adapter used by an evaluation run can be promoted into a worker without copying retry code into every call site.

## Preserve the quality-versus-latency decision

Rate-limit handling can quietly change model quality. If a queue falls back to a faster transcription configuration whenever it grows, the transcript distribution changes exactly when traffic is unusual. An evaluation harness should therefore score the policy, not merely the model: clean speech, warehouse noise, tracking numbers, carrier names, accented English, and clips with long silence should all pass through the same deadline and retry paths used in deployment.

A useful decision table is small enough to review with support operations:

| Queue state | Scheduling action | Quality action | Ticket action |
| --- | --- | --- | --- |
| Ready and within budget | Admit by priority | Use the evaluated transcription profile | Await transcript |
| Rate limited with usable delay | Reschedule for that delay | Keep the same profile | Show queued state |
| Rate limited without usable delay | Apply bounded jitter | Keep the same profile | Show queued state |
| Deadline reached | Stop automatic retries | Do not silently downgrade | Route audio for review |

The catch is that strict profile consistency is not suitable when a safety-critical ticket must be surfaced in seconds. In that case, use a separately evaluated low-latency path and mark its transcript provenance; don't invent an emergency fallback during an incident. Conversely, stick with the higher-quality path for routine proof-of-delivery questions when agents can tolerate queueing. The threshold belongs in a versioned policy with an owner, not in a magic conditional inside the API adapter.

Quality stays explicit.

I'm not sure which threshold is right for a given operation until its evaluation set includes real audio classes and its support team defines an acceptable age for each priority. That uncertainty is measurable. Replay a fixed corpus through candidate policies, then compare transcript task accuracy and end-to-end ticket age. For downstream RAG or agent classification, count prompt tokens independently from audio duration; the official `tiktoken` library is one available BPE tokenizer for that separate measurement. Prompt cost matters, but it should not be confused with transcription admission pressure.

## Batch admission without hiding overload

“Batch” should mean controlled admission of many independent ticket jobs unless the chosen speech interface explicitly defines request aggregation. Drain, for example, only the number of jobs for which the worker pool has capacity, while leaving the rest durably queued. A larger dequeue is not more throughput if it merely moves waiting work into process memory.

Keep urgent and routine work in distinct priority bands — with reserved capacity if the business requires it — so a morning import of delivery-confirmation recordings cannot starve a live damaged-goods report. Fairness still matters: age routine jobs upward or cap the urgent share, otherwise low-priority audio can wait forever during sustained load. Your mileage may vary because arrival shape, clip duration, and the provider's limits are deployment inputs, not universal constants.

Do not synchronize retries at the batch boundary. If 200 jobs receive the same missing delay and every worker uses the same deterministic backoff, they become a pulse. Per-job jitter spreads the next admission attempts, while the central scheduler continues to enforce concurrency. Tiny detail. Big effect.

An LLM gateway belongs after transcription only if the system uses one for ticket classification or summarization; it does not remove the need for speech admission control. LiteLLM is an open-source, self-hosted LLM gateway example, and that boundary is useful precisely because transcription queue health and downstream language-model health remain separate signals.

## Failure handling and observability

A 429 is a scheduling outcome, not proof that the audio is bad. Keep it out of permanent-failure counts. The adapter should classify authentication failures, invalid media, transport failures, and rate limits into distinct application outcomes, while the scheduler retries only outcomes its policy declares retryable. Redact ticket text and audio references from logs; use opaque job and trace identifiers for correlation.

Watch queue age by priority, ready-job count, delayed-job count, attempt count, deadline expirations, admission rate, completion rate, and transcription latency. The first two reveal customer impact earlier than an average API latency chart. Alerting on the 429 count alone is weak — a well-controlled queue may absorb a short limit period without breaching a ticket deadline, while a growing queue can hurt users before a dramatic error count appears.

Idempotency is the other half of retry safety. A worker can finish transcription and lose its lease before recording completion. Store a stable job identifier, make result attachment conditional on job state, and treat duplicate completion as a normal recovery path. Do the same for downstream summarization: a transcript should have a version, and any derived classification should record that version, so a corrected transcript can be reevaluated without guessing which prompt produced the current label.

## Operational handoff

Before deployment, run the evaluation corpus through success, explicit-delay, missing-delay, and deadline paths. Confirm that workers release capacity while a job is delayed, priority does not cause starvation, duplicate delivery does not duplicate ticket updates, and a process restart preserves scheduled work. Then load-test the application-side queue with a fake transport. A real speech endpoint is the wrong place to discover that the scheduler creates retry pulses.

During rollout, compare ticket age and task accuracy by priority and audio class. Keep queue-policy changes separate from model or prompt changes, because changing both destroys the comparison. The release is ready when on-call staff can answer four questions from telemetry: what is waiting, why is it waiting, when will it be eligible, and what happens when its deadline expires.

No queue policy removes a provider limit. It makes the response controlled, observable, and consistent with the logistics team's quality-versus-latency decision. That is the durable part of the design.

## Sources

- https://github.com/openai/tiktoken
- https://github.com/BerriAI/litellm
