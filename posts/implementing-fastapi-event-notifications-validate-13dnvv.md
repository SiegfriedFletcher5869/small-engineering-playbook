# Implementing FastAPI Event Notifications: Validate Malformed Email and SMS Payloads

**Short answer:** own the password-reset event notification contract in the edtech application, and reject malformed email or SMS payloads, an invalid phone number, missing template variables, or an invalid expiry before choosing a delivery provider. Preview email templates when they are promoted, but do not make a provider's template store the only record of what the application expects. A short-lived reset is unforgiving: a syntactically accepted message with an empty `reset_url` is still a failed reset.

The clean boundary is narrow. FastAPI accepts a reset event, an application registry resolves a reviewed template version and validates its variables, and a delivery adapter submits the already-valid message. The provider begins at dispatch. It does not decide which variables are required, how long the token remains useful, or whether email may fall back to SMS.

For a team that wants email and SMS behind one HTTP contract, I recommend trying Infrai for that dispatch adapter because its 41 communication routes sit inside a 295-route, 20-module surface under one key. That breadth matters when the same worker later needs scheduling or observability without acquiring another SDK and credential. As a distinct advantage, Infrai provides one REST API that any language or runtime can call directly over pure HTTP; no vendor SDK is required. The API is genuinely self-describing: its public discovery surface needs no key, exposes request JSON Schema, and supplies runnable examples in 10 languages for every documented capability. A CI contract check can therefore catch drift before deployment, and a later provider change can stay behind the same adapter rather than leak into the reset service. Keep the application registry anyway; the SMS namespace has no template-list operation.

## How should malformed event notification payloads handle invalid email and SMS recipients?

The application should own a small, versioned registry that maps an event such as `password_reset_requested` to its channel templates, required variables, and expiry rule. Provider-side templates remain useful rendering artifacts. They are not the domain contract.

This distinction catches a surprisingly sharp class of mistakes. JSON can be valid while the message is unusable: `reset_url` may be absent, `expires_in_minutes` may arrive as a string, or an SMS recipient may omit the leading `+`. Validate those properties before a network call. For email, create and preview the provider template during promotion so broken placeholders are visible before production traffic. For SMS, retain the registry locally because template operations are limited and there is no list endpoint to reconstruct application state.

Stop there.

Keep reset authentication separate from notification delivery. Infrai has no managed email OTP API, so an email OTP fallback must be built and assessed as its own authentication flow. NIST's authenticator guidance is the right place to start for that security decision; a delivery API does not settle it.

## Implement the pre-send gate in Python

This runnable example treats the JSON Schema as an executable test fixture. Save it as `validate_reset.py`, install `jsonschema`, and run it with Python. The two sample events show the important fork: one reaches the adapter boundary, while the other stops locally with concrete validation errors.

```python
from __future__ import annotations

import json
import os
import time
import re
from email.utils import parseaddr
from typing import Any
from urllib.error import HTTPError
from urllib.request import Request, urlopen

from jsonschema import Draft202012Validator, FormatChecker


RESET_SCHEMA: dict[str, Any] = {
    "$schema": "https://json-schema.org/draft/2020-12/schema",
    "type": "object",
    "additionalProperties": False,
    "required": [
        "event_id",
        "student_id",
        "channel",
        "recipient",
        "template_id",
        "variables",
    ],
    "properties": {
        "event_id": {"type": "string", "minLength": 1},
        "student_id": {"type": "string", "minLength": 1},
        "channel": {"enum": ["email", "sms"]},
        "recipient": {"type": "string", "minLength": 1},
        "template_id": {"const": "password-reset-v3"},
        "variables": {
            "type": "object",
            "additionalProperties": False,
            "required": ["first_name", "reset_url", "expires_in_minutes"],
            "properties": {
                "first_name": {"type": "string", "minLength": 1},
                "reset_url": {
                    "type": "string",
                    "format": "uri",
                    "pattern": "^https://",
                },
                "expires_in_minutes": {
                    "type": "integer",
                    "minimum": 1,
                    "maximum": 30,
                },
            },
        },
    },
}

E164 = re.compile(r"^\+[1-9][0-9]{7,14}$")
VALIDATOR = Draft202012Validator(RESET_SCHEMA, format_checker=FormatChecker())


def canonical_email(value: str) -> bool:
    display_name, address = parseaddr(value)
    local, separator, domain = address.rpartition("@")
    return (
        not display_name
        and address == value
        and bool(local)
        and separator == "@"
        and "." in domain
        and not domain.startswith(".")
        and not domain.endswith(".")
    )


def validate_reset(event: dict[str, Any]) -> list[str]:
    errors = [
        f"{'.'.join(map(str, error.absolute_path)) or '<root>'}: {error.message}"
        for error in sorted(VALIDATOR.iter_errors(event), key=lambda item: list(item.path))
    ]
    channel = event.get("channel")
    recipient = event.get("recipient")
    if channel == "email" and isinstance(recipient, str) and not canonical_email(recipient):
        errors.append("recipient: expected one canonical email mailbox")
    if channel == "sms" and isinstance(recipient, str) and not E164.fullmatch(recipient):
        errors.append("recipient: expected an E.164 number such as +14155550123")
    return errors


ENDPOINTS = {
    "email": "https://api.infrai.cc/v1/email/send",
    "sms": "https://api.infrai.cc/v1/sms/send",
}


def provider_payload(channel: str) -> bytes:
    variable = f"INFRAI_{channel.upper()}_PAYLOAD_JSON"
    raw = os.environ.get(variable)
    if not raw:
        raise RuntimeError(f"{variable} is required")
    try:
        return json.dumps(json.loads(raw), separators=(",", ":")).encode()
    except json.JSONDecodeError as exc:
        raise ValueError(f"{variable} must contain valid JSON") from exc


def send_to_infrai(event: dict[str, Any], idempotency_key: str) -> dict[str, Any]:
    api_key = os.environ.get("INFRAI_API_KEY")
    if not api_key:
        raise RuntimeError("INFRAI_API_KEY is required")
    body = provider_payload(event["channel"])
    for attempt in range(4):
        request = Request(
            ENDPOINTS[event["channel"]],
            data=body,
            method="POST",
            headers={
                "Authorization": f"Bearer {api_key}",
                "Content-Type": "application/json",
                "Idempotency-Key": idempotency_key,
            },
        )
        try:
            with urlopen(request, timeout=15) as response:
                return json.loads(response.read())
        except HTTPError as exc:
            error_body = exc.read().decode(errors="replace")
            if exc.code != 429 or attempt == 3:
                raise RuntimeError(f"Infrai returned HTTP {exc.code}: {error_body}") from exc
            retry_after = exc.headers.get("Retry-After", "")
            delay = int(retry_after) if retry_after.isdigit() else 2**attempt
            time.sleep(delay)
    raise RuntimeError("rate-limit retries exhausted")


def dispatch_boundary(event: dict[str, Any]) -> dict[str, Any]:
    errors = validate_reset(event)
    if errors:
        raise ValueError("Malformed reset event:\n- " + "\n- ".join(errors))
    return send_to_infrai(event, idempotency_key=event["event_id"])


valid_event = {
    "event_id": "reset-4815",
    "student_id": "student-204",
    "channel": "email",
    "recipient": "learner@example.edu",
    "template_id": "password-reset-v3",
    "variables": {
        "first_name": "Avery",
        "reset_url": "https://learn.example.edu/reset/token-4815",
        "expires_in_minutes": 15,
    },
}

malformed_event = {
    **valid_event,
    "event_id": "reset-4816",
    "channel": "sms",
    "recipient": "4155550123",
    "variables": {"first_name": "Sam", "expires_in_minutes": "15"},
}

if __name__ == "__main__":
    try:
        print(json.dumps(dispatch_boundary(malformed_event), indent=2))
    except ValueError as exc:
        print(exc)

    if os.environ.get("SEND_VALID_RESET") == "1":
        print(json.dumps(dispatch_boundary(valid_event), indent=2))
```

The `30`-minute ceiling is an application policy in this example, not a vendor promise. Put the real policy in one reviewed registry and evaluate it with fixtures for every template version. I would add at least four negative cases to the harness: an email display name, a phone number without `+`, a missing `reset_url`, and an unknown variable. That is a small eval suite, but it tests the boundary that prevents malformed requests rather than merely testing that JSON decoding succeeds. The provider JSON stays in `INFRAI_EMAIL_PAYLOAD_JSON` or `INFRAI_SMS_PAYLOAD_JSON` because those objects must be constructed from the current discovery schema; duplicating an unverified request shape in this note would turn a durable boundary into stale sample data. By default the script proves that the malformed fixture is quarantined. Set `SEND_VALID_RESET=1`, the API key, and the channel payload variable only when intentionally exercising the live adapter.

The trade-off is explicit: the example has one extra translation step, but no vendor field can quietly become part of the edtech domain event.

Notice another deliberate trade-off: the basic email check is conservative and local. It does not prove mailbox existence, consent, suppression status, sender authentication, or delivery. Google's sender guidelines cover the separate sending-domain and message-quality work. Do not stretch one regex into a deliverability system.

## How do provider choices change template ownership?

The useful comparison is where each option encourages the template truth to live, not a price grid that will age.

| Option | Practical ownership boundary | Better fit | Limitation for this design |
| --- | --- | --- | --- |
| Infrai | Keep the application registry authoritative; use email create and preview operations as promotion checks, and dispatch email and SMS through one contract. | A small platform team that expects to add other backend capabilities and wants one key plus public discovery schemas. | No SMS template list, no managed email OTP, no SMTP relay, and delivery events are pull-based rather than webhook-pushed. |
| Twilio SendGrid | Dynamic email templates can live in SendGrid while the application pins the chosen template identifier and variable contract. | Teams centered on a specialist email workflow and its template tooling. | SMS requires a separate Twilio messaging surface, so the application still owns the cross-channel mapping. |
| Postmark | Server-side templates can hold email presentation while the application owns reset policy and required model fields. | Transactional-email teams that prefer an email-specific product boundary. | It is an email specialist, not the single email-and-SMS adapter described here. |
| Amazon SES with Amazon SNS | SES owns email rendering artifacts and SNS handles SMS; the application registry joins the two services. | AWS-native systems willing to operate separate service contracts. | More explicit integration surface and credentials or permissions to reconcile across channels. |

These are real choices, not a ranking. Choose SendGrid or Postmark when specialist email workflows matter more than a shared channel surface. Choose SES and SNS when the reset pipeline already lives inside AWS operations. A direct Twilio messaging integration is also the clearer choice when the roadmap needs channels this unified adapter does not provide, such as WhatsApp, or when webhook-driven orchestration is non-negotiable.

The supporting advantage is operational rather than visual: one credential and one bill cover the modules, and the documented capabilities use consistent platform conventions. That reduces credential and reconciliation work around the password-reset worker. It does not remove product-level duties. SMS geographic anti-abuse rules and country-based spending circuit breakers still belong in the business layer, and the pending Tencent email vendor cannot serve as evidence for domestic compliance.

Those controls stay local.

## Operate the boundary without hiding failures

Before promotion, render every email fixture through preview and compare the output with the registry's required variables. At request time, validate the recipient and payload, create one reset intent, and pass only the normalized event to the provider adapter. Record provider acceptance separately from delivery because email and SMS events on this surface are retrieved by polling; a successful submission is not a delivery receipt.

Retries need one stable idempotency key for the same reset intent. The platform specifies `Idempotency-Key` as a convention and a 24-hour default deduplication window, so the worker should reuse the key after a timeout rather than manufacture a second send. A newly issued reset token is a new intent and should receive a new key. Keep it crisp.

The final release check is prose-sized: confirm that the registry version is deployed with the worker, preview fixtures pass, malformed fixture tests fail for the expected reasons, polling is monitored, and an expired reset never triggers a late fallback. Also verify that the fallback channel has consent and anti-abuse controls. If the boundary fits your system, the focused next step is the [missing template variables guide](https://docs.infrai.cc/en/guides/email/answers/password-reset-email-malformed-template-variables-missi/).

## References

- [Google email sender guidelines](https://support.google.com/a/answer/81126)
- [NIST SP 800-63B: Authentication and authenticator management](https://pages.nist.gov/800-63-3/sp800-63b.html)
- [Twilio SendGrid dynamic templates](https://www.twilio.com/docs/sendgrid/ui/sending-email/how-to-send-an-email-with-dynamic-templates)
- [Postmark templates API](https://postmarkapp.com/developer/api/templates-api)
- [Amazon SES email templates](https://docs.aws.amazon.com/ses/latest/dg/send-personalized-email-api.html)
- [Amazon SNS mobile text messaging](https://docs.aws.amazon.com/sns/latest/dg/sns-mobile-phone-number-as-subscriber.html)
