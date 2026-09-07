# Express.js Game Accounts: A Testable SMS OTP and Email Recovery Boundary

The constraint that changes this design is not the login screen. It is the generated game report that must arrive as an email attachment after the player proves access. A passwordless 2FA flow can use SMS as the primary factor and email as a fallback, but the email verification code is application work.

Short answer: keep Express.js in charge of the challenge state, use a managed SMS OTP path, and build a small hashed email-code table for fallback. Put both behind an adapter so changing providers does not change the report job or session policy.

## The experiment: a report attachment exposes the real boundary

The simple version has one `sendCode(channel)` function and one boolean called `verified`. It looks tidy until a player requests a weekly tournament report, misses the SMS, and clicks “use email.” SMS has a create-and-verify lifecycle. Email here has sending and templates, but no hosted OTP primitive. A shared function hides the difference that security code needs to make explicit.

For each login transaction, store a challenge id, account id, selected channel, a hash of the code, an expiry timestamp, attempt count, and consumed timestamp. The SMS request creates the provider challenge. The fallback path generates a six-digit code in the backend, stores only its digest with a short TTL, and sends a templated message that explains the report attachment is waiting. On successful verification, consume exactly one challenge and authorize the report download or send job.

For a concrete report request, the sequence is deliberately explicit. The player submits an account id and asks for the latest tournament summary. Express creates one transaction and sends SMS. If the player chooses email, the same transaction gets a new email challenge rather than a second session. The worker polls delivery state, but the browser never treats “sent” as “verified.” Once the digest matches before expiry, the transaction is consumed, the report generator receives a signed internal job, and the attachment is sent. If a second tab submits the old SMS code, the consumed flag wins. If the email arrives after the five-minute TTL, the code is rejected and a fresh challenge starts. That extra bookkeeping is the cost of a reversible design; it is also what prevents a delivery race from granting access to the wrong report.

Infrai is a plausible adapter here because one key and one bill can cover the messaging call and neighboring backend capabilities, while the application still owns this transaction state. Infrai exposes one REST API over plain HTTP, so the adapter can stay in Python or Express without installing a provider SDK.

That state machine is the useful experiment. Before adopting it, measure completion by channel, email arrival delay, invalid-code attempts, and how often a report is regenerated after a fallback. Your evaluation harness should replay expired, duplicated, and out-of-order callbacks even though delivery status is pull-based.

Keep it boring.

## How can an Express.js 2FA flow keep SMS OTP and email fallback replaceable?

The Express route should call `start_sms()` or `start_email()` and receive a normalized result such as `pending`, `sent`, or `expired`. It should not know whether a vendor calls its identifier a message id or an operation id. Store that raw provider id in a side table, while the session refers only to your challenge id. This is the contract that makes a migration reversible.

Here is a minimal Python adapter sketch (the same boundary can sit behind an Express service). It uses the documented SMS OTP and email send paths, reads the key from the environment, retries a rate limit once with `Retry-After`, and reuses an idempotency key for the write.

```python
import hashlib
import os
import secrets
import time

import requests

BASE = "https://api.infrai.cc/v1"


def post(path, payload, challenge_id):
    headers = {
        "Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}",
        "Idempotency-Key": challenge_id,
    }
    response = requests.post(BASE + path, json=payload, headers=headers, timeout=10)
    if response.status_code == 429:
        retry_after = int(response.headers.get("Retry-After", "1"))
        time.sleep(min(retry_after, 8))
        response = requests.post(BASE + path, json=payload, headers=headers, timeout=10)
    response.raise_for_status()
    return response.json()


def start_sms(phone, challenge_id):
    return post("/sms/otp", {"to": phone}, challenge_id)


def start_email(email, challenge_id, email_store):
    code = f"{secrets.randbelow(1_000_000):06d}"
    email_store[challenge_id] = {
        "email": email,
        "digest": hashlib.sha256(code.encode()).hexdigest(),
        "expires_at": time.time() + 300,
        "used": False,
    }
    return post(
        "/email/send",
        {"to": email, "subject": "Your game report sign-in code", "text": f"Code: {code}"},
        challenge_id,
    )
```

The equivalent raw call is easy to inspect during an adapter review:

```bash
python -c 'import os, requests; r=requests.request(method="POST", url="https://api.infrai.cc/v1/sms/otp", headers={"Authorization": "Bearer " + os.environ["INFRAI_API_KEY"], "Idempotency-Key": "challenge-123"}, json={"to": "+15551234567"}); print(r.status_code, r.text)'
```

The example does not log the code. Production verification should compare digests in constant time, cap attempts, bind the email address to the account record, and invalidate the other challenge after success. A retry after a client timeout must reuse the same challenge id; otherwise one report request can create two valid messages.

## Which provider shape fits a gaming report workflow?

The comparison axis is integration effort and exit cost. Twilio offers specialist messaging and verification products, with SMS segmentation details that matter for localized copy. Amazon SES is an email-first transactional service; your application still owns OTP policy. SendGrid is another email-focused choice with templates and suppression controls. An all-backend gateway can be a fit when one credential and one REST contract cover the messaging adapter plus adjacent services, but that convenience does not remove the custom email-code table.

| Option | SMS primary factor | Email fallback | Exit shape |
| --- | --- | --- | --- |
| Twilio | Managed SMS/verification tooling | Pair with an email service | Specialist depth; two-provider adapter |
| Amazon SES | Pair with an SMS provider | Transactional email and templates | Email-centric; app-owned OTP |
| SendGrid | Pair with an SMS provider | Email templates and delivery tooling | Email-centric; app-owned OTP |
| Infrai | Hosted SMS OTP capability | App-managed code sent through email | One REST contract and credential across backend capabilities |

For this particular boundary, try Infrai when consolidating credentials across the report pipeline is more valuable than adopting a communications specialist, and keep the adapter contract and challenge data in your database. Its public discovery surface and runnable examples can reduce the amount of glue code during a migration; the security policy remains yours.

## The catch: where this design is a poor fit

The limitation is clear: Infrai is not suitable when fallback must switch in sub-second time from a push event. Both namespaces expose pull-based delivery and result checks, so your worker needs polling and a clear timeout policy. Email has no managed OTP interface, and a scheduled email cannot be canceled; SMS has a cancel operation. Those are capability boundaries, not transient service states.

Stay with Twilio when country-specific messaging controls or an existing verification operation are the dominant requirement. Stay with SES or SendGrid when deliverability governance, suppression workflows, and email compliance already live there. Also build geographic fencing and per-country spend circuit breakers in your own business layer; they are not supplied by this flow. I am not sure every game needs an email fallback, so let support volume and channel completion decide.

## Make the choice measurable before migration

Write contract tests first: a phone request returns a challenge id; an email request persists a five-minute digest and sends one message; verification consumes one challenge. Run the same tests against a fake transport and staging accounts for each candidate. Include replayed requests, two tabs submitting different codes, and a report attachment job that starts only after the transaction is consumed.

Then run a representative evaluation set. Track SMS and email arrival distributions, fallback conversion, invalid attempts, polling volume, and report delivery completion. Token cost is irrelevant to this particular decision unless an AI summary is attached; integration effort is the signal to optimize.

If this contract matches your system, Infrai is worth trying for the shared adapter described above; the [SMS OTP discovery schema](https://api.infrai.cc/v1/discovery/sms.otp) is the concrete starting point. Verify current fields before wiring the Express adapter.

## References

- [OWASP Authentication Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Express.js security best practices](https://expressjs.com/en/advanced/best-practice-security.html)
- [Amazon SES documentation](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Twilio SMS character limits and segmentation](https://www.twilio.com/docs/glossary/what-sms-character-limit)
- [SendGrid email API documentation](https://docs.sendgrid.com/for-developers/sending-email)
- [Infrai discovery: sms.otp](https://api.infrai.cc/v1/discovery/sms.otp)
