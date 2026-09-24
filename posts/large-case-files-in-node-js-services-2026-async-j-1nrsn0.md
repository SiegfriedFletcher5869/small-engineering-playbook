# Large Case Files in Node.js Services: 2026 Async Jobs, Retries, and Retention

Property managers rarely fail because a watermark is hard to draw. They fail when a 300-page case file is uploaded twice, a retry keeps a private copy forever, or a reviewer receives a file before the watermark job has actually finished. **Short answer:** treat watermarking as a durable asynchronous job with an idempotency key, validate every input before processing, keep temporary bytes isolated and encrypted, and make deletion a measured workflow rather than a cron hope.

I build retrieval and agent features in Python, so my first instinct is to make a notebook prove the transformation. That is useful, but a notebook does not model a queue redelivery or a tenant deleting a case while a worker is busy. The production design has to make those transitions explicit.

## A property-management flow that survives a large file

The concrete flow is an external-share request for a property case file: lease scans, inspection photos, invoices, and correspondence are assembled, watermarked, and delivered to an outside adjuster. The API should acknowledge the request quickly with a job identifier. A queue consumer then reads immutable input metadata, streams the source into a private workspace, applies the template owned by the property-management team, writes a new object, and records a manifest. Download authorization checks that manifest, not the worker's local state.

The template ownership choice matters. If legal or compliance owns the watermark template, the rendering service should accept a versioned template ID and refuse silent edits. If an operations team owns it, give them a review workflow and an audit event for each activation. In either case, the case file, template version, and output hash belong in the job record so a later retry cannot produce an unexplained variant.

Here is a compact worker sketch. The queue, object store, and renderer are deliberately generic; the important part is the state machine and cleanup behavior.

```python
from dataclasses import dataclass
from datetime import datetime, timedelta, timezone
from pathlib import Path
import hashlib
import os
import secrets
import tempfile


@dataclass(frozen=True)
class Job:
    job_id: str
    tenant_id: str
    source_key: str
    template_id: str
    expires_at: datetime


def process_watermark_job(job: Job, store, renderer, ledger):
    now = datetime.now(timezone.utc)
    if now >= job.expires_at:
        ledger.expire(job.job_id, reason="retention_deadline")
        return

    # A unique directory limits accidental cross-tenant reads during retries.
    with tempfile.TemporaryDirectory(prefix=f"case-{job.tenant_id}-") as work:
        source = Path(work) / "source.bin"
        output = Path(work) / "watermarked.bin"
        store.download_to_file(job.source_key, source, decrypt=True)

        validate_case_file(source, max_bytes=2_000_000_000)
        template = ledger.active_template(job.template_id)
        renderer.apply(source, output, template=template)

        digest = hashlib.sha256(output.read_bytes()).hexdigest()
        destination = f"{job.tenant_id}/exports/{job.job_id}.pdf"
        store.put_if_absent(destination, output, metadata={"sha256": digest})
        ledger.complete(job.job_id, output_key=destination, output_sha256=digest)


def validate_case_file(path: Path, max_bytes: int):
    size = path.stat().st_size
    if size == 0 or size > max_bytes:
        raise ValueError("file size outside policy")
    with path.open("rb") as handle:
        magic = handle.read(5)
    if magic != b"%PDF-":
        raise ValueError("unexpected document type")
```

`put_if_absent` is the small detail that keeps a redelivered message from creating a second output. In a real service, `read_bytes()` should be replaced by a streaming hash for multi-gigabyte files; the example keeps that line visible because it makes the manifest contract easy to inspect. My eval harness checks that the same `(tenant_id, source hash, template version)` produces the same output hash, while a changed template produces a new job version.

## How should a service handle asynchronous jobs, retries, validation, and temporary files?

Start with a finite state machine: `queued`, `running`, `succeeded`, `failed`, and `expired`. Store an attempt count, next-attempt timestamp, and a reason code. A worker claims a job with a lease. If its lease expires, the queue may redeliver the message; the completion write must therefore be conditional on the job version still being current. This is more reliable than trying to make the queue exactly once.

Retries need categories. A transient object-store timeout can use exponential backoff with jitter, capped by a deadline. A malformed PDF, an unauthorized template, or a failed policy check is permanent and should move to `failed` without retry. Keep the original exception in restricted logs, but expose only a stable reason code to the requester. This prevents personal data from leaking into dashboards and makes alert counts useful.

That distinction is the boundary.

When a retry is allowed, persist the decision before publishing the next message so a process crash cannot reset the attempt count. Include a deterministic idempotency token in the output key, and compare the source object's content hash before reusing a prior result; a matching filename is not evidence that two uploads are the same. During a long render, renew the worker lease at a cadence shorter than the queue visibility timeout, but stop renewing once the job deadline or legal hold policy says processing must end. This sequence gives operators a visible explanation for every delayed case file, even when the original request has already timed out in the browser.

Validation happens twice. Before enqueueing, reject impossible sizes, unsupported media types, and missing tenant ownership. In the worker, inspect the actual bytes, page count, and template version again because object metadata can be stale or the source can be replaced between requests. Malware scanning belongs at the trust boundary; rendering should run with no network access and a low-privilege filesystem account.

Temporary files deserve the same policy as permanent exports. Use an encrypted volume or an encrypted filesystem, generate an unguessable directory per attempt, set restrictive permissions, and delete the directory in a `finally` path. A process crash can skip that path, so a sweeper must remove abandoned workspaces by age and emit a deletion metric. Do not put source bytes, signed URLs, or full filenames in queue payloads.

The browser side has a related trap: a `Blob` can keep a large in-memory copy alive after a download appears complete. Revoke object URLs after use and prefer streaming responses for files that exceed the client memory budget. The MDN Blob documentation describes the object URL lifecycle and is a useful review reference.

## Privacy and retention are executable controls

Write a retention policy as data: source deadline, output deadline, audit-event deadline, and legal-hold flag. For example, an external-share output might expire 72 hours after successful delivery, while an audit record survives longer under the organization's records schedule. The numbers are policy decisions, not defaults to hide in a worker constant. Your service should calculate `expires_at` at job creation and enforce it at download, processing, and cleanup time.

Deletion needs evidence. Record which object key was deleted, when, by which service identity, and whether a legal hold blocked the action. Tombstone the job metadata so an old queue message cannot resurrect the bytes. Backups and replicas need their own expiry process; deleting the primary object is not proof that every copy is gone. I'm not sure every storage backend offers the same deletion guarantees, so the retention review should include provider-specific documentation and a restore test.

Measure it.

Consider a tenant who asks for deletion while an adjuster is downloading a finished export. The authorization layer should check the tombstone before issuing a new signed URL, while an already-issued URL should have a short lifetime and be invalidated when the object is removed. The worker needs the same check: if the source is under a deletion request, it should stop before fetching another chunk and mark the job `expired`, even if a queue redelivery arrives with an old payload. The sweeper then handles the workspace, the source copy, and the output according to their separate deadlines. An audit event can say that deletion was requested, blocked by a legal hold, or completed; it should never include the lease text, tenant address, or a link that grants access. On restore, the service must reapply expiry metadata rather than quietly bringing an old export back into the download namespace. This is a longer path than deleting one database row, but it is the path that makes a privacy promise testable under concurrency.

Keep identifiers pseudonymous in telemetry. A metric such as `watermark_job_duration_seconds{template_version="v17"}` is useful; a metric labeled with an address or tenant name is not. Sample payload-level traces only in a quarantined test environment, and make redaction part of the test suite. Privacy is an operational property that needs assertions, not a paragraph in a policy document.

## Choosing boundaries and testing the unpleasant paths

The renderer should be replaceable behind a narrow interface. A self-hosted library may give tighter data residency control but transfers patching and font-management work to your team. A hosted document API can reduce operations work while introducing a data-transfer review and an availability dependency. A queue backed by your existing platform is easier to observe; a specialized workflow engine can model long waits and human approvals better. None of these choices removes the need for idempotency or retention enforcement.

My test matrix starts with a 1-page fixture and then jumps to a 10,000-page synthetic case file. It injects a timeout after upload, a worker kill after rendering, a duplicate message, a template revocation, and a tenant deletion request during processing. Assertions check that at most one output is authorized, no expired output downloads, and every temporary directory disappears after the sweeper window. I also run a prompt-cost check on the metadata extraction step: watermarking should not send the entire case file to an AI model when a page count and a hash are enough.

The catch is operational ownership. This pattern is not suitable when a team cannot run a queue, encrypted workspace, and deletion monitor; a managed workflow may be the better boundary then. Stick with a simpler synchronous path for small, non-sensitive documents where the extra state machine would create more failure surface than value. For large property case files shared outside the company, the explicit machinery usually pays for itself in auditability.

Before shipping, walk the checklist as prose: confirm the template owner and version policy, set an idempotency key, classify retryable errors, validate bytes in the worker, isolate and encrypt the workspace, enforce expiry on every read path, test crash cleanup, and review what appears in logs. Then run the eval harness against a duplicate delivery and a legal hold. Those two cases expose more real defects than another happy-path screenshot.

## References

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.rfc-editor.org/rfc/rfc9110
- https://owasp.org/www-community/attacks/Path_Traversal

## Sources

- https://developer.mozilla.org/en-US/docs/Web/API/Blob
- https://www.rfc-editor.org/rfc/rfc9110
- https://owasp.org/www-community/attacks/Path_Traversal
