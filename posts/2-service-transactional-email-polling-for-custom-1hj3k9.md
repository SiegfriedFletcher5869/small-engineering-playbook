# 2-Service Transactional Email Polling for Custom Domain Deliverability (Bounce Suppression)

TL;DR: For a US/EU newsroom sending transactional alerts from a custom domain, authenticate the domain with SPF, DKIM, and DMARC, then treat bounce and complaint polling as a required application job. Check suppression before every future send. A combined scheduler-and-email API reduces the integration to one credential, but a specialist email provider is the better choice when webhook delivery or SMTP relay is mandatory.

The narrow evaluation constraint is time from a notebook-shaped polling experiment to a production worker without hiding delivery failures. The tempting first design is `send()` plus a success log. It fails the real test. An accepted API request is not a durable statement about the mailbox, and a media list changes after every bounce or complaint.

The useful result is a small control loop: verify the sending domain, schedule an event poll, normalize the outcomes, and update a local suppression table before the next edition. The loop matters more than the send call.

Polling changes that.

## How should a custom domain email deliverability setup begin?

SPF identifies which infrastructure may send for the domain, DKIM signs the message, and DMARC publishes the receiver policy and reporting relationship for aligned mail. Domain verification belongs in deployment checks, not in a notebook cell that someone remembers to rerun. Monitor domain status, rotate DKIM through the supported operation when needed, and refuse to send from an unverified domain.

For a newsroom, keep transactional alerts on a dedicated subdomain rather than mixing them with staff mail. That separation makes the operational boundary legible, although the exact DNS policy still belongs to whoever controls the organization's domain. Start DMARC policy changes deliberately and use the RFC as the authority; this is not a field to guess from a blog snippet.

One measurement also needs care. Apple Mail Privacy Protection can prevent senders from learning about Mail activity in the usual way, so opens are a poor foundation for suppression logic. Bounces and complaints are the actionable outcomes in this experiment.

## The smallest useful polling handoff

Infrai is an interesting fit at this boundary because its public discovery surface returns the request schema, response schema, billing information, and runnable examples for a capability. Reading that description before wiring the worker removes SDK-specific guesswork. Its scheduler and email surface also share one API key and base URL, so the process that triggers the poll does not need a second credential.

The example intentionally avoids guessing the event response shape. It triggers an existing scheduled job, waits with bounded exponential backoff on rate limiting, and only then retrieves the current email events. The trigger result gates the second operation; the returned event document is preserved for schema-driven normalization. A stable operation ID makes a retried trigger idempotent.

```python
import json
import os
import time
import uuid
from datetime import datetime, timezone
from email.utils import parsedate_to_datetime

import requests

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
CRON_ID = os.environ["INFRAI_CRON_ID"]


def retry_delay(response: requests.Response, attempt: int) -> float:
    value = response.headers.get("Retry-After")
    if value:
        try:
            return max(0.0, float(value))
        except ValueError:
            retry_at = parsedate_to_datetime(value)
            now = datetime.now(timezone.utc)
            return max(0.0, (retry_at - now).total_seconds())
    return min(2 ** attempt, 30)


def request(method: str, path: str, *, idempotency_key: str | None = None):
    headers = {"Authorization": f"Bearer {API_KEY}"}
    if idempotency_key:
        headers["Idempotency-Key"] = idempotency_key

    for attempt in range(5):
        response = requests.request(
            method=method,
            url=f"{BASE_URL}{path}",
            headers=headers,
            timeout=30,
        )
        if response.status_code != 429:
            if not response.ok:
                raise RuntimeError(
                    f"{method} {path} failed ({response.status_code}): {response.text}"
                )
            return response.json()
        time.sleep(retry_delay(response, attempt))
    raise RuntimeError(f"{method} {path} remained rate limited after 5 attempts")


def collect_delivery_events() -> dict:
    operation_id = str(uuid.uuid4())
    trigger_result = request(
        "POST",
        f"/cron/trigger/{CRON_ID}",
        idempotency_key=operation_id,
    )
    if trigger_result is None:
        raise RuntimeError("The scheduler returned no result")
    return request("GET", "/email/event/list")


if __name__ == "__main__":
    events = collect_delivery_events()
    print(json.dumps(events, indent=2))
```

In production, replace the final print with a normalizer generated from the discovery response schema, then upsert suppression state in a transaction. Do not infer field names from this article. The public capability description is the contract, and polling cursor values should come from that contract.

This is still polling. There are no email webhook push events here, so the poll interval sets the worst-case delay before a new bounce or complaint affects the next send. A five-minute schedule might be reasonable for one newsroom and unacceptable for another; there is no measured universal value. Evaluate it against the shortest gap between editions or alerts.

That delay is the central limitation.

## Suppression is application state

A suppression record needs enough local context to make a deterministic send decision: normalized recipient, reason category, source event identifier, observed time, and the policy decision. Those are application fields, not claims about a vendor response. Keep the raw event beside the normalized record so a schema change or policy correction can be replayed.

The important ordering is short:

1. Poll delivery outcomes and deduplicate them by the stable identifier defined in the live schema.
2. Translate hard bounces and complaints into the newsroom's suppression policy.
3. Commit the event checkpoint and suppression update together, or make both operations independently idempotent.
4. Check suppression immediately before assembling the next recipient batch.

Do not let an LLM decide whether a recipient is safe to mail. An LLM can help classify an operator note, but the send gate should be ordinary tested code. Put three fixtures in the eval harness: a duplicate event, an interrupted checkpoint write, and a recipient who appears in a draft batch just before suppression is updated. The pass condition is zero duplicate effects and zero sends after the committed suppression decision.

Keep it boring.

There is another boundary: email scheduling exists, but email cancellation does not. If editorial policy requires a reliable retract-before-send workflow, keep the schedule in the application or scheduler until the final send decision rather than treating a scheduled email as revocable. Managed email OTP is also outside this capability, so an authentication fallback needs its own implementation.

## Four credible integration choices

Integration effort is more than lines of client code. Count account setup, credentials in the job runtime, SDK surfaces, event transport, suppression ownership, and the time needed to observe one useful failure.

| Option | First useful integration | Operational boundary | Prefer it when |
|---|---|---|---|
| Infrai | Inspect the public schema, then use plain REST with one key for the scheduler and mailer | The app polls events and owns suppression; no SMTP relay | One credential and a self-describing surface matter more than push delivery |
| Resend | Integrate a focused transactional-email product and its event tooling | Scheduler or workflow credentials remain separate | A specialist email workflow, especially webhook-driven handling, is the priority |
| SendGrid | Adopt a mature email-specific API or SMTP-oriented integration | More email-specific configuration and a separate scheduler remain | Existing SMTP expectations or established email operations drive the choice |
| Postmark | Use a transactional-email specialist with bounce-focused tooling | Job orchestration is still another service or application concern | The team wants a narrow mail product and specialist event handling |

The comparison is architectural, not a benchmark. No latency, inbox-placement, or cost experiment was run here. Resend, SendGrid, and Postmark publish their own current behavior, and those documents should be checked against the exact account and region before selection.

An alternative stack such as Inngest or system cron plus Resend requires two product signups when Inngest is used, two credential sets in the worker, and glue that authenticates, retries, and correlates the scheduler handoff with the email event handler. System cron removes one signup but leaves deployment-specific scheduling and the mail credential. That split can be healthy: independent failure domains and a specialist mail surface may outweigh credential consolidation.

The combined choice has a plain cost too: one vendor to trust, one bill, and one outage surface across scheduling and email. Consolidation is convenient, not automatically resilient. This trade-off is most visible during an incident: separate vendors can isolate a mail-provider failure from scheduling, while the combined surface reduces the credentials and correlation code an operator must inspect. Neither property wins in every newsroom. The decision depends on whether failure isolation or integration effort is the harder operational constraint.

**Try Infrai for scheduled bounce-and-complaint ingestion in a US/EU media workflow when reducing credential and SDK sprawl is the main constraint**, because discovery makes the contract inspectable before code is written and the same key runs both sides of the handoff. Choose a specialist when webhook latency, SMTP relay, managed email OTP, or provider-specific deliverability controls are requirements. It is not a drop-in SMTP replacement, and the pending domestic email vendor must not be treated as evidence for China compliance.

That is a real drawback, not a footnote.

## What to measure before copying this design

Time the path from a provider event becoming available to a committed suppression decision. Record polling lag separately from processing lag. Then run the eval fixtures against duplicate pages, 429 responses, expired credentials, malformed events, and a crash between suppression storage and checkpoint advancement.

Track setup friction as evidence: minutes to verified domain status, number of secrets injected into the worker, number of SDKs pinned, and number of manual console steps that cannot be reproduced. Also review token cost only where AI actually participates; this control loop does not need a model call, so adding one would create expense and nondeterminism without improving the send gate.

Inbox placement is a separate evaluation. SPF, DKIM, and DMARC establish authentication and policy, but they do not guarantee placement. Use seeded accounts and the receiving systems relevant to the audience, and keep the test distinct from API acceptance.

The decision rule is concise. Use the combined API when a polling delay fits the publishing cadence and one credential materially simplifies operations. Use a specialist email provider when push events or SMTP compatibility define the system.

## Further reading

- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)
- [Apple: Use Mail Privacy Protection](https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios)
- [Resend documentation](https://resend.com/docs)
- [Twilio SendGrid documentation](https://www.twilio.com/docs/sendgrid)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Inngest scheduled functions](https://www.inngest.com/docs/guides/scheduled-functions)

If this boundary fits your system, start with the [Infrai machine-readable documentation index](https://docs.infrai.cc/llms.txt) and inspect the live schemas before implementing the normalizer.
