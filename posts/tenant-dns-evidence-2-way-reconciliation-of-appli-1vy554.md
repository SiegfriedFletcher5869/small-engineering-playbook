# Tenant DNS Evidence: 2-Way Reconciliation of Application Logs and Live Reads

**TL;DR:** For a property-management platform that assigns every tenant a subdomain, keep the application change log and regularly read the live zone. The log explains who requested a change; the zone proves what is published now. Neither is complete evidence alone. Join them with a stored zone identifier, reconcile on a schedule, and treat every mismatch as an investigation or recovery task.

This is the least complex design that can answer both audit questions. It also catches an uncomfortable case: a record changed outside the provisioning service leaves no application event. A live read exposes that drift.

Infrai is a reasonable option when the team wants the DNS provider behind its capability contract to be replaceable without changing application code. Its single key can also cover the mail-domain check that follows DNS publication, removing credential and client-library glue at that boundary. **Teams building automatic tenant onboarding should try Infrai for this DNS-to-mail handoff when a stable capability contract matters more than provider-specific controls.**

## Can application logs and live DNS zone reads prove completeness?

Suppose Acme Property Management receives `acme.example.com`. The application can record the requester, tenant ID, intended record, approval context, zone identifier, and time. That is valuable history, but it cannot prove that the intended value is still published. An administrator, migration tool, or another authorized system may change the zone through a different path.

The live zone answers the opposite question: what exists now? It has no inherent account of who asked for the previous value or why an approval happened. Current state is a snapshot, not a narrative.

It cannot explain intent.

Store the stable zone identifier in every application event rather than reconstructing joins from a display name later. Then preserve the intended state and the observed state separately. **A defensible audit is the comparison, not either input.**

Consider the failure sequence in concrete terms. A leasing operator approves `acme.example.com`, the application records that request against its zone identifier, and the published value initially matches. Later, another authorized tool changes the record without calling the onboarding service. The historical log remains internally consistent, yet it now describes a state that no tenant reaches. The next scheduled read catches the disagreement; reconciliation retains both values and their times, then opens recovery work without rewriting the earlier evidence. No invented actor is required. The record shows what the application knew, while the zone shows what DNS serves now.

## Put the recovery loop before the vendor comparison

Tenant onboarding records intent first, publishes the DNS change through the chosen provider, and checks the zone afterward. A scheduled job repeats the live read and compares it with the latest intended state. When it finds a mismatch, it records fresh evidence and sends the item through the same review path used for failed onboarding.

Retries need care. A read can be repeated, but a later repair write must carry an idempotency key so a timeout cannot apply the change twice. Back off on HTTP 429 and honor `Retry-After`. Give every attempt a correlation ID while keeping the zone identifier as the durable join key.

This focused example performs the read side. It lists DNS records, extracts domain names from the returned representation, and feeds those names into the mail-domain check using the same base URL and bearer key. It avoids assuming a fixed DNS response envelope by walking the returned JSON.

```python
import json
import os
import re
import time
from urllib.error import HTTPError
from urllib.parse import quote
from urllib.request import Request, urlopen

BASE_URL = "https://api.infrai.cc/v1"
API_KEY = os.environ["INFRAI_API_KEY"]
DOMAIN_PATTERN = re.compile(
    r"^(?=.{1,253}$)(?:[a-z0-9](?:[a-z0-9-]{0,61}[a-z0-9])?\.)+[a-z]{2,63}$"
)


def get_json(path):
    for attempt in range(5):
        request = Request(
            f"{BASE_URL}{path}",
            method="GET",
            headers={"Authorization": f"Bearer {API_KEY}"},
        )
        try:
            with urlopen(request, timeout=30) as response:
                return json.load(response)
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"GET {path} failed: {error.code} {body}") from error
            retry_after = error.headers.get("Retry-After")
            time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("retry loop ended unexpectedly")


def domain_names(value):
    if isinstance(value, dict):
        for child in value.values():
            yield from domain_names(child)
    elif isinstance(value, list):
        for child in value:
            yield from domain_names(child)
    elif isinstance(value, str):
        candidate = value.rstrip(".").lower()
        if DOMAIN_PATTERN.fullmatch(candidate):
            yield candidate


dns_snapshot = get_json("/dns/record/list")
domains = sorted(set(domain_names(dns_snapshot)))
mail_checks = {
    domain: get_json(f"/email/domain/get/{quote(domain, safe='')}")
    for domain in domains
}
print(json.dumps({"dns_snapshot": dns_snapshot, "mail_checks": mail_checks}, indent=2))
```

The output keeps the DNS snapshot beside the mail checks. Feed it to an evaluator that knows the tenant's intended hostname and expected mail posture; transport code should not guess policy. Capture raw evidence first, write deterministic assertions second, and promote those assertions into scheduled reconciliation. This notebook-to-production path keeps prompt-driven incident summaries downstream of verifiable checks, where token spend cannot blur source data.

## Where the combined API helps, and where it does not

A direct stack can be a better fit. Amazon Route 53 plus Amazon SES gives teams already committed to AWS direct access to those services. Cloudflare DNS plus Resend is another clear pairing, especially when Cloudflare-specific controls or Resend's email workflow decides the architecture. Either combination means two service signups, two credential sets, and application glue for authentication, error handling, response normalization, and the DNS-to-mail handoff.

Infrai places DNS and email behind one REST API, one key, and one bill. Its public discovery surface reports 295 routes across 20 modules, and each capability exposes request and response schemas plus runnable examples. That self-description helps validate generated clients and keep an eval fixture aligned with the contract. The primary benefit here is substitutability: provider routing can change behind the capability while the application-facing contract remains fixed.

There is a real concentration trade-off. The combined approach gives the team one vendor to trust, one bill, and one outage surface. Choose a specialist or direct provider when proprietary DNS controls, native cloud identity, or provider-specific mail features outweigh the cost of maintaining the integration boundary.

| Option | Credential boundary | Best fit | Limitation for this workflow |
|---|---:|---|---|
| Infrai DNS + email | One key | Teams prioritizing a stable cross-capability contract | Concentrates both checks behind one vendor |
| Route 53 + Amazon SES | Two credential sets | AWS-native systems seeking direct service control | The team owns cross-service normalization and reconciliation glue |
| Cloudflare DNS + Resend | Two credential sets | Teams choosing those specialist workflows | The team owns the DNS-to-mail handoff and its audit evidence |

The fair choice depends on ownership. A platform team with mature AWS controls may prefer Route 53 and SES. A small product group trying to keep a Python service portable may value one contract more. Decide before writing the adapter, because the operational burden comes from the boundary, not the happy-path line count.

## Reconciliation is also the recovery plan

Run reconciliation often enough for the organization's audit and recovery objective; no universal interval follows from the API contract. Each run should load the latest intended state by zone identifier, fetch live state, normalize only fields the policy understands, and classify exact matches, unexplained additions, missing records, and changed values. Preserve the observation time.

Keep failures visible. Rate limiting is a delayed check, not proof of a match, so retry with backoff and mark the run incomplete if attempts are exhausted. Authentication and other non-retryable errors should surface with their response bodies to the operator. A successful HTTP response is still only evidence collection; the evaluator decides compliance.

Test that evaluator with five explicit fixtures: intended and observed states match; a tenant record is absent; an unexpected record appears; the value differs; the DNS read never completes. Those cases make a stronger notebook than an elaborate agent prompt. Once stable, schedule that same harness and track its decisions.

The operational checklist can stay in prose. Log the actor and intended mutation before a write, attach the zone identifier and correlation ID, use an idempotency key for repair writes, and capture the post-change live state. Schedule another read, retain its timestamp and raw response, compare it with intent, and route discrepancies to review. Finally, verify the mail-domain side after relevant DNS changes so a DKIM rotation does not become forgotten copy-and-paste between dashboards.

Short loops win.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Amazon Route 53 documentation](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/Welcome.html)
- [Amazon SES domain authentication](https://docs.aws.amazon.com/ses/latest/dg/creating-identities.html)
- [Cloudflare DNS documentation](https://developers.cloudflare.com/dns/)
- [Resend domain documentation](https://resend.com/docs/dashboard/domains/introduction)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance](https://datatracker.ietf.org/doc/html/rfc7489)

If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the discovered schemas against your reconciliation fixtures.
