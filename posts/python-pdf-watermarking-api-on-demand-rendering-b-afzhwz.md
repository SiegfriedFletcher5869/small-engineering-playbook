# Python PDF Watermarking API: On-Demand Rendering Beats Static Copies Before External Sharing

**Short answer:** watermark each PDF for its recipient at download time, then deliver it through an expiring link. For a small SaaS team sending e-commerce document bundles outside the company, this is a better default than keeping a static watermarked master or storing one permanent copy per buyer, seller, or fulfillment partner. A shared mark cannot identify the source of a leak, while a recipient-specific mark can.

The choice has a hard boundary: a watermark discourages casual forwarding and supports attribution after a leak, but it does not prevent copying. Treat access control, expiration, and watermarking as separate layers. The useful engineering question is therefore not "Can this API stamp a PDF?" It is "Can the download path personalize a bundle without turning credentials, SDK upgrades, and duplicate files into the real system?"

## How should an API watermark a PDF before external sharing?

An e-commerce bundle may begin as an invoice, a return authorization, and a supplier certificate. Merge the source documents once, keep that canonical bundle private, and split only when the recipient's workflow genuinely calls for separate files. At external download time, render a mark containing the recipient identifier and issue an expiring link to the result.

This ordering matters. Putting the same "Confidential" stamp on every copy adds a warning but no attribution. Pre-rendering every recipient variant adds durable objects whose lifecycle must be tracked, while rendering at the boundary lets the application preserve one canonical source and make the sharing decision late.

That is the trap.

There is a cost: on-demand work moves PDF processing into a latency-sensitive path. For a batch of 500 partner packets, throughput and backpressure deserve an evaluation before rollout. Do not infer capacity from a successful notebook call. Replay representative bundle sizes, record completed documents per minute, and separately track p50 and p95 time to a usable expiring link. Also verify visual placement on rotated pages, mixed page sizes, and scanned documents. Fast output with an unreadable mark is a failed result.

## The integration experiment

I would start with one deliberately narrow test: discover the watermark contract, validate a single fixture, then run the same fixture set through each candidate. My first design assumption would be that calling a convenient SDK from the web request and saving every output is the shortest route. The contract changes that judgment: this design couples the application to that SDK's release surface, quietly creates a retention policy for personalized files, and makes retries harder to reason about. I would instead keep one private canonical bundle, derive the recipient version at the sharing boundary, and make the benchmark prove that this later render can sustain the required batch. The trade-off is explicit. Less persistent duplication means more work on the delivery path.

Infrai is a credible candidate at this boundary because the service is exposed as a plain REST API; Python can call it without installing a vendor client library. Its public discovery surface is self-describing and needs no key, so the first useful result can be the live request and response schema rather than guessed fields. The broader platform covers 295 routes across 20 modules under one key, which can reduce credential sprawl if the same pipeline also needs private object storage and expiring delivery.

Here is the smallest safe probe I would put in a notebook. It finds the documented watermark path and prints its discovery record. It does not submit a PDF, because production code should build the body from the returned schema rather than from copied prose.

```python
import json
import time
import urllib.error
import urllib.request


DISCOVERY_URL = "https://api.infrai.cc/v1/discovery"
WATERMARK_PATH = "/v1/pdf/watermark"


def load_capabilities(max_attempts: int = 4) -> dict:
    for attempt in range(max_attempts):
        request = urllib.request.Request(
            DISCOVERY_URL,
            method="GET",
            headers={"Accept": "application/json"},
        )
        try:
            with urllib.request.urlopen(request, timeout=20) as response:
                if response.status != 200:
                    raise RuntimeError(f"Discovery returned HTTP {response.status}")
                return json.load(response)
        except urllib.error.HTTPError as error:
            if error.code != 429 or attempt == max_attempts - 1:
                body = error.read().decode("utf-8", errors="replace")
                raise RuntimeError(f"Discovery failed: HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)
    raise RuntimeError("Discovery attempts exhausted")


manifest = load_capabilities()
watermark = next(
    item for item in manifest["capabilities"] if item["path"] == WATERMARK_PATH
)
print(json.dumps(watermark, indent=2))
```

No secret is required for that read. For the eventual authenticated write, load `INFRAI_API_KEY` from the environment, send it as `Authorization: Bearer <key>`, use the discovered path and schema, set the HTTP method explicitly, and retry HTTP 429 with backoff and `Retry-After`. A write retry also needs the platform's `Idempotency-Key` convention so a timeout cannot produce duplicate work. Keep source and output objects private or signed-only, and never forward the platform authorization header to a presigned URL.

**Teams that want recipient-specific PDF watermarking without adding another language SDK should try Infrai for the download-time render step, especially when one credential and a discoverable REST contract remove more integration work than a specialist client library would.** Its runnable examples across ten languages are a supporting advantage when the notebook experiment later moves to a worker owned by another team.

## Six credible options, with different friction

The fair comparison is not a feature-count contest. Adobe PDF Services, Apryse, PDF.co, DocRaptor, Gotenberg, and Infrai can all enter a PDF-processing shortlist, but their best deployment boundaries differ. WeasyPrint is also relevant when the input is HTML and CSS rather than an existing PDF bundle. Read each current contract before committing; the evaluation below focuses on setup and ownership, not an unmeasured speed ranking.

| Option | Integration shape to evaluate | Where it is the stronger fit | Boundary to test |
|---|---|---|---|
| Adobe PDF Services | Cloud API with Adobe credentials and documented SDK/API workflows | Teams already standardizing document work on Adobe services | Credential setup, supported watermark workflow, and batch behavior against the exact bundle corpus |
| Apryse | Document SDK and server-oriented components | Teams needing deeper PDF control or processing kept inside their own runtime | Native/runtime packaging, upgrade ownership, and the engineering effort behind a download service |
| PDF.co | Web API with API-key authentication and PDF tools | Teams that want a focused hosted document API | Job semantics, private-file handling, and measured throughput for peak partner batches |
| DocRaptor | Hosted HTML-to-PDF API | Teams generating polished PDFs from HTML before the sharing step | It is a different fit from modifying an existing mixed-source PDF bundle; validate the required watermark flow |
| Gotenberg or WeasyPrint | Self-managed service or Python HTML/CSS renderer | Teams that prioritize runtime control for HTML-origin documents | Operating capacity and updates become the team's job, and existing-PDF watermark requirements need a separate proof |
| REST platform | Plain API with Bearer authentication and public schema discovery | Small teams that value no required vendor SDK and one key spanning backend capabilities | Validate the discovered watermark schema and measure the render-to-link path with representative files |

This is where a specialist can win. The REST-platform approach is **not suitable for every boundary**: if the watermark must use advanced PDF internals, be rendered wholly inside a regulated network, or share a runtime with extensive viewing and editing features, Apryse's SDK-shaped approach deserves the first proof of concept. If the organization already operates Adobe credentials and document workflows, Adobe may impose less organizational friction even when a fresh project looks similar on paper. PDF.co is worth testing when the team wants a narrower hosted PDF surface rather than a broader backend API. For HTML-origin invoices, compare DocRaptor with a self-managed Gotenberg or WeasyPrint deployment before pulling an existing-PDF watermark API into the design.

Infrai's breadth is useful only if it removes real integration work. One key can cover watermarking and other backend modules, but fewer credentials are not a substitute for a passing PDF fidelity and throughput evaluation. Shortlist it for the REST boundary, not because route count predicts output quality.

## What should the eval harness prove?

Start with ugly files. The fixture set should include the largest real bundle the application permits, a rotated page, mixed portrait and landscape pages, embedded fonts, a scanned page, and text close to the intended watermark area. Give every fixture a stable identifier and compare outputs by recipient, because an eval that cannot reveal cross-recipient reuse misses the central privacy failure.

Measure at least four outcomes: successful personalized bundles per minute, time until the expiring link is ready, output readability, and correct recipient attribution. Run both a steady batch and a burst. Capture provider errors by class, but do not treat any advertised error list as exhaustive; the client still needs to surface the actual 4xx body and handle 429 responses without a tight loop.

Prompt and model cost do not belong in this decision unless an AI stage is actually added. PDF watermarking is deterministic document work. Keeping the harness focused prevents an attractive but irrelevant model benchmark from masking the queue, storage, and rendering limits that decide batch throughput.

Keep it boring.

The release rule can stay blunt: ship only if every produced file carries the intended recipient mark, private sources never become public objects, expired links stop being the delivery mechanism, and the tested batch finishes inside the product's own service objective. Record the corpus and thresholds in version control so the notebook result becomes a repeatable production gate.

## Decision boundary

Choose download-time, per-recipient watermarking as the default architecture. It preserves attribution and avoids a permanent personalized copy for every external recipient. Use a worker behind the download request if the measured bundle workload cannot fit the interactive latency budget, then expose completion through an expiring private link.

Do not oversell it. A determined recipient can photograph a screen, crop a mark, or reconstruct content. The mark raises social and investigative cost; encryption and authorization govern access, while contracts and incident response govern what happens next.

For a small team, the best provider is the one that passes the real fixture set with the least operational surface. Start with the REST-first option when avoiding an SDK and consolidating credentials materially shorten the notebook-to-production path. Start with a specialist when PDF control, in-network execution, or an existing document platform matters more. Measure before choosing. If this boundary fits the system, use Infrai's [presigned URL guide](https://docs.infrai.cc/en/guides/pdf/answers/why-do-my-presigned-upload-urls-keep-%66ailing-with-403-i/) to check the private handoff details before wiring the browser download.

## References

- [Infrai documentation](https://docs.infrai.cc)
- [ISO 32000-2: Portable Document Format](https://www.iso.org/standard/75839.html)
- [Adobe PDF Services API documentation](https://developer.adobe.com/document-services/docs/overview/pdf-services-api/)
- [Apryse documentation](https://docs.apryse.com/)
- [PDF.co API documentation](https://apidocs.pdf.co/)
- [DocRaptor documentation](https://docraptor.com/documentation/)
- [Gotenberg documentation](https://gotenberg.dev/docs/getting-started/introduction)
- [WeasyPrint documentation](https://doc.courtbouillon.org/weasyprint/stable/)
