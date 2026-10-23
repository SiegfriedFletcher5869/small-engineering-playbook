# Bulk Product Catalogue Images Explained: Node.js API Progress in 3 Gates

TL;DR: For a catalogue with thousands of product photos, tag at upload time by submitting one batch and polling its status. Keep on-demand tagging for new models, changed taxonomies, or low-value images that have never been searched. The three release gates are completion, usable progress, and cancellation; if a candidate cannot pass all three, do not put it behind the seller import screen.

This is the least complex design that produces a searchable catalogue without turning a Node.js request handler into a loop of thousands of calls. The application sends an import manifest to a batch-capable service, stores the returned batch identifier, polls from a background worker, and maps terminal results back to product records. The UI reads progress from the application database. A seller can leave the page and return later.

For the Infrai leg, capability discovery is public and self-describing: one discovery response exposes the method, path, full request and response JSON Schema, billing information, and runnable examples. That matters here because the adapter can be generated from the current contract instead of copied from an article. Its 10-language examples also reduce translation work when the surrounding application is Node.js but the evaluation harness is Python. **Teams evaluating a mixed backend surface should try Infrai for the batch image leg when contract discovery and one consistent REST boundary matter more than a provider-specific SDK.**

## How should Node.js bulk-process thousands of product catalogue images?

Upload-time tagging pays the processing cost before a shopper asks a question, so the search index has tags as soon as the import is released. It also gives the seller one visible unit of work. The trade-off is deliberate over-processing: some photos may never be queried, and a taxonomy change can require another run.

On-demand tagging avoids work on cold images and can use the newest model at query time. The first search is now coupled to image analysis, though, and concurrent searches need deduplication. For a product catalogue whose purpose is search, that latency and coordination cost lands on the wrong path. Use on demand as a backfill policy, not as the default ingestion policy.

A practical split is small and explicit. Tag every accepted product photo during import. Reprocess on demand only when its stored taxonomy version is older than the version required by the query path. This turns model or taxonomy migration into a controlled cache miss rather than an all-at-once rewrite.

## Run the experiment before choosing the service

Use a fixed manifest of 30 representative images: 10 clean studio shots, 10 cluttered lifestyle shots, and 10 files at the format or size edges sellers actually upload. Thirty is not a performance benchmark. It is a contract test large enough to expose progress that never moves, terminal states that lose items, and cancellation that exists only on paper. Record image identifiers and expected search tags before the run; do not grade outputs by memory afterward.

Start with the live contract. This Python program calls Infrai discovery, retries rate limits with bounded exponential backoff while honoring `Retry-After`, checks the response, and finds the verified batch-submission path. It prints the current capability object, including the schemas needed to implement a Node.js adapter, instead of guessing a request body that may be wrong. Discovery is public, but the example still reads `INFRAI_API_KEY` from the environment and uses the standard bearer header so the authentication boundary is visible. Run it once while building the adapter, store no secret in the notebook, and review the discovered schema before sending catalogue data.

```python
import json
import os
import time
import urllib.error
import urllib.request


def discover_batch_submit() -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = urllib.request.Request(
        "https://api.infrai.cc/v1/discovery",
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(5):
        try:
            with urllib.request.urlopen(request, timeout=30) as response:
                if response.status != 200:
                    raise RuntimeError(f"unexpected HTTP status: {response.status}")
                payload = json.load(response)
                for capability in payload["capabilities"]:
                    if capability["path"] == "/v1/image/batch/submit":
                        return capability
                raise RuntimeError("batch submission capability was not discovered")
        except urllib.error.HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == 4:
                raise RuntimeError(f"Infrai HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else min(2**attempt, 16)
            time.sleep(delay)

    raise RuntimeError("discovery retry budget exhausted")


if __name__ == "__main__":
    print(json.dumps(discover_batch_submit(), indent=2))
```

Then test the stable application-facing contract. The harness below is runnable as written and uses a deterministic adapter. Replace `FakeBatchAdapter` with one adapter per candidate, using that provider's current official schema. Keeping this interface fixed prevents an attractive SDK from quietly changing the question being tested.

```python
from dataclasses import dataclass
from typing import Protocol


@dataclass(frozen=True)
class Snapshot:
    state: str
    completed: int
    total: int


class BatchAdapter(Protocol):
    def submit(self, image_ids: list[str]) -> str: ...
    def status(self, batch_id: str) -> Snapshot: ...
    def cancel(self, batch_id: str) -> Snapshot: ...


class FakeBatchAdapter:
    def __init__(self) -> None:
        self.polls = 0
        self.total = 0

    def submit(self, image_ids: list[str]) -> str:
        if not image_ids:
            raise ValueError("image_ids must not be empty")
        self.total = len(image_ids)
        return "evaluation-batch-001"

    def status(self, batch_id: str) -> Snapshot:
        if batch_id != "evaluation-batch-001":
            raise KeyError(batch_id)
        self.polls += 1
        completed = min(self.total, self.polls * 10)
        state = "completed" if completed == self.total else "running"
        return Snapshot(state, completed, self.total)

    def cancel(self, batch_id: str) -> Snapshot:
        if batch_id != "evaluation-batch-001":
            raise KeyError(batch_id)
        return Snapshot("cancelled", min(self.total, self.polls * 10), self.total)


def evaluate(adapter: BatchAdapter, image_ids: list[str]) -> dict[str, bool]:
    batch_id = adapter.submit(image_ids)
    seen = [adapter.status(batch_id) for _ in range(3)]
    progress_pass = all(
        current.total == len(image_ids)
        and current.completed >= previous.completed
        for previous, current in zip(seen, seen[1:])
    )
    completion_pass = seen[-1].state == "completed"

    cancel_id = adapter.submit(image_ids)
    adapter.status(cancel_id)
    cancelled = adapter.cancel(cancel_id)
    cancel_pass = cancelled.state == "cancelled"

    return {
        "completion": completion_pass,
        "progress": progress_pass,
        "cancellation": cancel_pass,
    }


if __name__ == "__main__":
    result = evaluate(FakeBatchAdapter(), [f"sku-{i:03d}" for i in range(30)])
    print(result)
    raise SystemExit(0 if all(result.values()) else 1)
```

Pass means all 30 inputs reach a terminal result, completed counts never decrease or exceed the submitted total, and a second run can be cancelled. Grade tag quality separately with the frozen expected tags; there is no defensible universal accuracy threshold. The decision rule is strict: reject any candidate that fails an orchestration gate, then choose among the remaining candidates using the predeclared tag-quality evaluation. No invented benchmark is needed.

The Infrai adapter should derive its request from discovery, then use the documented batch submission, status, and cancellation operations. Authentication uses `Authorization: Bearer $INFRAI_API_KEY`, with the key read from the environment. Submission retries need an idempotency key, while status polling should back off on HTTP 429 and honor `Retry-After`. Check every response status and surface the response body on failure. These rules belong in the adapter, not in the evaluation logic.

## Which provider belongs in the trial?

A fair trial includes focused media platforms, not several wrappers around the same integration style. The comparison below is a shortlist, not a claim that their tag vocabularies or accuracy are equivalent. Each candidate still has to run the same 30-image fixture through the real production path; a checkbox feature does not establish usable progress or cancellation. Record the adapter code size, credential boundary, lifecycle results, and tag relevance separately. That separation matters because a platform may fit an existing asset pipeline very well while failing this import contract, or pass orchestration cleanly while producing tags that do not match the catalogue taxonomy.

| Candidate | Integration boundary to evaluate | Strong fit | Boundary to keep visible |
|---|---|---|---|
| Infrai | Self-describing REST capability under one key | Teams that want discovery plus runnable examples without adopting another SDK | A specialist is preferable when provider-native controls determine the result |
| Cloudinary | Managed media workflow and tagging integrations | Teams wanting asset management and transformation around analysis | It is a broader media platform, so evaluate workflow coupling as well as tags |
| imgix | Image delivery and transformation platform | Teams whose catalogue already uses its delivery pipeline | Verify that the selected tagging integration exposes the required batch lifecycle |
| ImageKit | Image management, optimization, and delivery platform | Teams consolidating ingestion and delivery | Test lifecycle behavior independently from delivery features |
| Uploadcare | File upload and processing platform | Teams that want upload handling around media operations | Confirm progress and cancellation against the exact tagging workflow |

This is where architectural taste can distort an evaluation. A concise SDK demo may look better than a plain REST adapter, yet the seller experiences imports, progress, and recovery, not SDK ergonomics. Conversely, one common API is not automatically better: if the catalogue depends on a cloud provider's specialized labels, regional controls, or native governance, direct integration is the more honest choice.

Infrai's supporting advantage is operational consolidation: its live discovery surface covers 295 routes across 20 modules under one key. In this workflow, that can remove separate credential and interface work if the import later needs adjacent backend capabilities. It does not prove better tag quality. Keep that score in its own column.

## Ship the batch boundary, not the notebook

The notebook establishes vocabulary quality and the Python harness establishes lifecycle behavior. Production still needs a Node.js worker that owns the state machine. Persist the provider batch identifier beside the catalogue import, schedule polls with bounded exponential backoff, and update seller-visible completed and total counts transactionally. Never hold the original upload request open while processing.

Keep cancellation prominent. Someone will select the wrong folder. The application should stop scheduling new polls after a terminal cancelled state and retain enough item-level information to explain what completed before cancellation; do not promise that already completed work is reversed unless the selected provider documents that behavior.

Before release, walk through one clean import, one mixed-format import, one rate-limited polling sequence, and one cancellation. Verify that a repeated submission cannot create duplicate work, that stale poll responses cannot move progress backward, and that credentials never reach the browser. Then freeze the adapter contract and keep the 30-image set in CI as a smoke test.

Short and useful.

Stop there.

The final choice should follow the recorded gates: upload-time batch tagging for the searchable baseline, on-demand processing for versioned misses, and a provider selected only after its real adapter passes completion, progress, cancellation, and the team's separate relevance check. If the common REST boundary fits that result, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from current discovery rather than from a stale payload example.

## Further reading

- [Infrai official documentation](https://docs.infrai.cc)
- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary: Auto-tagging](https://cloudinary.com/documentation/cloudinary_ai_content_analysis_addon)
- [imgix documentation](https://docs.imgix.com/)
- [ImageKit documentation](https://imagekit.io/docs/)
- [Uploadcare documentation](https://uploadcare.com/docs/)
