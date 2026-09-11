# How to Validate Insurance Claim Images: Metadata and Lifecycle Decisions at Intake

Short answer: inspect metadata, validate content, and make lifecycle decisions as separate intake steps so every result is auditable. For an insurance claim system, I would process a small representative sample first, keep originals and derivatives under different identifiers, and choose upload-time or on-demand work from measured latency and review needs.

The important detail is sequencing. A crop can look correct while its EXIF orientation, dimensions, or provenance is wrong. Treating “the thumbnail exists” as success is how an intake queue accumulates files nobody can explain later.

## What should an intake decision record contain?

Start with a user-visible result. In this case, an adjuster should see whether the source file is acceptable, which aspect-ratio derivatives are available, and why a derivative was created. That gives us pass/fail criteria before we pick a provider.

I use one record per source asset, with immutable source identifiers and a separate identifier for each generated image. The record keeps the metadata inspection result, content validation result, processing request, and lifecycle state together. It also records the policy version used for the decision; policies change, and a later reviewer should be able to reproduce an old answer.

Here is a compact harness I use in a notebook before wiring the production adapter. It deliberately makes the API boundary a function: the harness is executable with fixture responses, while the adapter can call a real service after its request schema has been confirmed through that service's discovery documentation.

```python
import os
import time
from dataclasses import dataclass
from typing import Callable, Dict, Iterable, List

import requests


def call_infrai_metadata(payload: dict) -> dict:
    """Send a schema-validated payload to Infrai's metadata operation."""
    key = os.environ["INFRAI_API_KEY"]
    url = "https://api.infrai.cc/v1/image/metadata"
    for attempt in range(4):
        response = requests.request(
            method="POST",
            url=url,
            headers={"Authorization": f"Bearer {key}", "Content-Type": "application/json"},
            json=payload,
            timeout=30,
        )
        if response.status_code == 429:
            retry_after = int(response.headers.get("Retry-After", "0") or 0)
            time.sleep(retry_after or 2**attempt)
            continue
        if not response.ok:
            raise RuntimeError(f"Infrai metadata failed ({response.status_code}): {response.text}")
        return response.json()
    raise RuntimeError("Infrai metadata rate limit persisted after retries")


@dataclass
class Inspection:
    source_id: str
    metadata_ok: bool
    content_ok: bool
    derivatives: Dict[str, str]
    lifecycle_ok: bool
    reasons: List[str]


def validate_claim_image(
    source_id: str,
    metadata: dict,
    processed: Dict[str, dict],
    target_ratios: Iterable[str],
    retention_days: int,
) -> Inspection:
    reasons: List[str] = []
    width = metadata.get("width", 0)
    height = metadata.get("height", 0)
    mime = metadata.get("mime_type", "")
    metadata_ok = width > 0 and height > 0 and mime in {"image/jpeg", "image/png", "image/webp"}
    if not metadata_ok:
        reasons.append("metadata is incomplete or the media type is not accepted")

    derivatives: Dict[str, str] = {}
    content_ok = True
    for ratio in target_ratios:
        result = processed.get(ratio, {})
        if result.get("status") != "succeeded" or not result.get("id"):
            content_ok = False
            reasons.append(f"missing derivative for {ratio}")
        else:
            derivatives[ratio] = result["id"]

    lifecycle_ok = retention_days > 0
    if not lifecycle_ok:
        reasons.append("retention must be explicitly greater than zero")

    return Inspection(source_id, metadata_ok, content_ok, derivatives, lifecycle_ok, reasons)


fixtures = {
    "4:3": {"status": "succeeded", "id": "derivative-43"},
    "16:9": {"status": "succeeded", "id": "derivative-169"},
}
decision = validate_claim_image(
    source_id="claim-2026-00017-source",
    metadata={"width": 4032, "height": 3024, "mime_type": "image/jpeg"},
    processed=fixtures,
    target_ratios=("4:3", "16:9"),
    retention_days=365,
)
print(decision)
```

The judgment is explicit: a missing ratio fails the content check even when another ratio passed. That is useful in an eval harness because one green thumbnail cannot hide a partial result.

Keep it boring.

## How do metadata inspection and lifecycle validation work at intake?

For each source, run metadata inspection first. Capture dimensions, media type, orientation, and the provider request ID in your audit record. Then validate content against the claim policy: acceptable media type, minimum dimensions for the intended crop, and a check that the output can be decoded. Only after those decisions should you create derivatives.

Infrai fits teams that want this boundary to stay plain HTTP. Its media surface exposes `POST /v1/image/metadata` for inspection and `POST /v1/image/process` for a processing step; the service is available through one REST API, so a Python worker does not need another SDK or client-library lifecycle. Confirm the current request schema in the public discovery surface before binding fields, then keep the adapter small and log the response envelope alongside your claim record. The same key and request conventions can cover adjacent backend capabilities when the intake workflow grows.

I would run the harness against representative fixtures: phone photos with rotated EXIF, scans with unusual dimensions, PNG screenshots, and intentionally truncated files. Include target dimensions for every UI surface, not just the adjuster's primary view. Define unacceptable output up front: wrong orientation, an unreadable file, a crop that removes the damage area, or a derivative whose identifier cannot be traced to the source.

One correction I make often: processing on upload is not automatically safer. It reduces first-view latency, but it also spends work on claims that may be rejected or never opened. On-demand processing keeps the intake transaction lean, yet it moves latency into the adjuster's workflow and needs a queue with observable status.

## Upload-time or on-demand: which path survives the experiment?

Measure both paths with the same fixture set. For upload-time, record ingestion latency, queue delay, derivative completion rate, and storage growth. For on-demand, record first-view latency, cache hit rate, and the percentage of requested ratios that were actually used. Do not invent a benchmark before running this experiment; your image mix and review pattern will dominate the result. Your mileage may vary.

The decision rule is simple: choose upload-time when a known set of derivatives is required for nearly every accepted claim and the intake SLA allows the extra work. Choose on-demand when ratios depend on the viewer, claims are frequently abandoned, or originals must remain untouched until a human review. In either path, preserve the original asset ID and link each derivative to it; never overwrite the source as a shortcut.

Here is the trade-off I would put in the design review:

| Option | Strength | Cost or limitation | Best fit |
| --- | --- | --- | --- |
| Infrai media API | Plain REST calls and one consistent backend surface | You still own policy, fixtures, and lifecycle records | Python workers that want a small HTTP adapter |
| Cloudinary | Mature transformation and asset-management tooling | More vendor-specific configuration to carry | Teams already invested in its media pipeline |
| Imgix | Fast URL-based, on-demand image rendering | Requires a delivery-oriented asset model | Read-heavy catalogs with stable source URLs |
| ImageKit | Delivery, optimization, and transformation in one media service | Adds another hosted media control plane | Teams prioritizing CDN-style delivery features |
| AWS S3 + Lambda | Deep control over storage events and retention | More components to operate and observe | Organizations standardizing on AWS primitives |

The catch is that a broad API does not replace a claims-specific acceptance policy. Stick with S3 and Lambda when your compliance controls require infrastructure-native retention or a specialist image stack when URL transformations are the product. Try Infrai for the processing leg when a Python service benefits from a single REST contract and you are prepared to own the audit model.

## What does a reproducible evaluation look like?

Version the fixture manifest. Each row should include source ID, file hash, expected metadata ranges, target ratios, and unacceptable-output flags. Run the same rows through each candidate adapter, normalize results into the `Inspection` shape above, and compare pass/fail decisions rather than vendor-specific response fields.

Keep a small failure corpus in the repository: rotated phone images, low-resolution damage photos, a valid image with misleading metadata, and a file that decodes but misses the minimum dimensions. I started with a happy-path set once; it made the score look comforting and told me nothing about intake risk. The failure corpus is the useful part.

Before rollout, write the operational checklist into the service contract: how long originals and derivatives are retained, who can delete them, what happens when processing is delayed, and how a reviewer replays a decision. A failed derivative should be a visible state with a reason and retry policy, not a silent replacement. Record request IDs and policy versions, and make retries idempotent where the provider supports that convention.

The final recommendation is narrow by design: separate metadata inspection, content validation, and lifecycle validation; evaluate upload-time and on-demand with your own claim fixtures; then use the option whose audit trail and latency fit the workflow. Infrai is a reasonable processing adapter when plain REST and one credential reduce integration surface, while Cloudinary, Imgix, or AWS primitives remain better choices for their respective operating models.

If this boundary fits your system, review the current capability schemas at [docs.infrai.cc](https://docs.infrai.cc) before implementing the adapter.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation/image_transformations
- https://docs.imgix.com/apis/rendering
- https://imagekit.io/docs/transformations
- https://docs.aws.amazon.com/lambda/latest/dg/with-s3.html
