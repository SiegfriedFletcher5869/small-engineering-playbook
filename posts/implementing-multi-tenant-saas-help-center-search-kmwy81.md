# Implementing Multi-Tenant SaaS Help Center Search: One Collection or Many

A marketplace change watcher pays an index-cost penalty every time unchanged boilerplate becomes a new vector. That constraint changes the design choice. **TL;DR: start with one collection, make `tenant_id` a mandatory pre-filter, and index only normalized page versions whose content hash changed.** Move a tenant to a separate collection only when its isolation, retention, embedding, or operational requirements genuinely diverge. Collection count alone is not a security boundary.

This is an experiment note, not a universal sizing rule. The simple approach is to scrape on a schedule and insert every result. It fails quietly: navigation timestamps and tracking parameters produce duplicate embeddings, while a missing tenant predicate can return another customer's help content. The chosen approach puts deterministic change detection before embedding and tests isolation as an invariant. For a marketplace watching seller policy and shipping pages, that trims avoidable indexing work and makes alerts explainable.

That is the trap.

## Why doesn't one collection mean mixed-up tenants?

A shared collection mixes storage, not authorization. Each chunk must carry a stable tenant identifier, canonical page identifier, source revision, and content hash. Every retrieval path must apply the tenant predicate before results can influence an answer or alert. Filtering after nearest-neighbor retrieval is the wrong boundary: an unauthorized chunk has already entered the candidate set, and a small post-filtered set can also hurt recall.

The application still has to authorize the caller before it constructs that predicate. Treat the tenant value from a browser or webhook as untrusted input; derive the effective tenant from authenticated server-side context. The index filter then enforces scope again. Two checks are deliberate here.

Per-tenant collections do reduce the consequence of one missing metadata filter, but they create another source of mistakes: collection routing. They also multiply schema migrations, retention jobs, backups, health checks, and small underfilled indexes. A shared collection is the better starting experiment when tenants use the same chunking and embedding policy and the retrieval engine can apply metadata constraints during search. Separate collections become justified for contractual physical isolation, tenant-specific retention or encryption boundaries, incompatible embedding spaces, or a tenant large enough to need an independent capacity and recovery plan.

## Put change detection before retrieval

HTTP already provides useful inputs. A watcher can retain `ETag` and `Last-Modified`, then send conditional requests with `If-None-Match` or `If-Modified-Since`; a `304 Not Modified` response means there is no representation body to process. Those validators save transfer and parsing work when a server supports them, but they do not replace content hashing. A weak or absent validator, a changed template, or semantically identical markup can still require local judgment. The relevant HTTP semantics are specified in RFC 9110 and conditional request behavior in RFC 9111.

Normalize only what the alert contract declares irrelevant. Removing scripts, collapsing whitespace, and resolving links may be appropriate for a help article. Deleting all numbers would be disastrous for a seller-fee page because the number may be the change. Keep both the fetched revision metadata and the normalized text hash so an alert can show why a version was accepted.

The focused prototype below uses standard-library interfaces and an in-memory index seam. Replace `fetch` and `store` at deployment time, but preserve the order of operations.

```python
from __future__ import annotations

from dataclasses import dataclass
from hashlib import sha256
from html.parser import HTMLParser
from typing import Callable, Iterable, Protocol
from urllib.parse import urldefrag


class TextExtractor(HTMLParser):
    def __init__(self) -> None:
        super().__init__()
        self.parts: list[str] = []
        self.hidden_depth = 0

    def handle_starttag(self, tag: str, attrs: list[tuple[str, str | None]]) -> None:
        if tag in {"script", "style"}:
            self.hidden_depth += 1

    def handle_endtag(self, tag: str) -> None:
        if tag in {"script", "style"} and self.hidden_depth:
            self.hidden_depth -= 1

    def handle_data(self, data: str) -> None:
        if not self.hidden_depth:
            self.parts.append(data)


def normalize_html(html: str) -> str:
    parser = TextExtractor()
    parser.feed(html)
    return " ".join(" ".join(parser.parts).split())


@dataclass(frozen=True)
class PageVersion:
    tenant_id: str
    canonical_url: str
    content_hash: str
    text: str


class VersionStore(Protocol):
    def latest_hash(self, tenant_id: str, canonical_url: str) -> str | None: ...

    def index(self, version: PageVersion) -> None: ...


def accept_changed_page(
    tenant_id: str, url: str, html: str, store: VersionStore
) -> PageVersion | None:
    canonical_url, _fragment = urldefrag(url)
    text = normalize_html(html)
    digest = sha256(text.encode("utf-8")).hexdigest()
    if store.latest_hash(tenant_id, canonical_url) == digest:
        return None

    version = PageVersion(tenant_id, canonical_url, digest, text)
    store.index(version)
    return version


def scoped_search(
    tenant_id: str,
    query: str,
    search: Callable[..., Iterable[PageVersion]],
) -> list[PageVersion]:
    # The adapter must translate this into an engine-side pre-filter.
    return list(search(query=query, where={"tenant_id": tenant_id}))
```

The short code is intentional. Embedding, chunking, and alert delivery sit behind the acceptance decision; unchanged content never reaches them. In a notebook I would begin with fake `VersionStore` and `search` implementations, because they make the two important assertions cheap: the second identical page produces no version, and tenant A can never observe tenant B. Production adapters should keep the same tests.

## Make isolation and alert quality executable

An evaluation set for this system needs more than question-answer relevance. Build fixtures around marketplace changes: a seller return window changes from 30 to 14 days; a navigation timestamp changes while policy text remains stable; a page disappears; two tenants publish identical text; a page changes and then reverts. Label whether each event should create a version, an embedding, and an alert.

Start with four gates:

1. Cross-tenant retrieval count is exactly zero for every adversarial query.
2. An unchanged normalized page creates zero new indexed chunks.
3. A labeled material change creates one traceable version before any alert is sent.
4. Deletion and access revocation remove the page from retrieval within the service's declared freshness objective.

Zero means zero. Aggregate precision can hide a single isolation failure, so the first gate must not be averaged with relevance scores. For alert quality, track false alerts separately from missed material changes; collapsing them into one score conceals the product trade-off. A marketplace operator may tolerate a delayed low-impact wording alert but cannot accept another tenant's private help article as evidence.

Retrieval-augmented generation combines retrieved external memory with generation, as described in the original RAG paper. That architecture makes retrieval scope part of answer correctness. The generator cannot repair an authorization mistake upstream, and a polished response can make leaked context harder to notice. Log the tenant scope, canonical page ID, source hash, accepted revision, and retrieved chunk IDs for each alert decision. Avoid logging full private content by default.

## Model index cost with changed chunks, not tenant count

A collection-per-tenant spreadsheet often starts with the wrong variable. The useful workload estimate is closer to:

```python
def daily_embedding_units(
    pages_checked: int,
    changed_fraction: float,
    chunks_per_changed_page: float,
    retries_per_chunk: float = 0.0,
) -> float:
    return (
        pages_checked
        * changed_fraction
        * chunks_per_changed_page
        * (1.0 + retries_per_chunk)
    )
```

This is a dimensionless planning model, not a price calculator. Measure each input from replay traffic: pages checked, fraction with meaningful normalized changes, chunks produced per accepted version, retry amplification, retained versions, and query volume. Then compare shared and separated layouts under the same workload. Include fixed operational work such as migrations and backup verification, because thousands of mostly empty collections can be cheap in vectors yet expensive to operate.

Prompt cost belongs in the same evaluation harness. A diff alert usually needs the changed passages and limited surrounding context, not every chunk from both page versions. Generate a structured diff first, retrieve the minimum evidence needed to explain it, and cap context by an explicit token budget. This keeps the indexing decision connected to the downstream job instead of optimizing vector count in isolation.

Do not infer a universal break-even number. Engine behavior, filtering strategy, index type, tenant size distribution, and retention policy determine it. Run a shadow replay with the largest tenants and the noisiest pages. Record pre-filter recall, query latency percentiles, index growth, embedding units, alert precision, missed-change rate, and recovery time. Copy the one-collection choice only if isolation remains perfect and those measurements stay inside your own objectives.

Measure it first.

## Promotion and escape criteria

Ship the notebook logic as a deterministic ingestion stage, then add queues and idempotency around it. A retry must use a key such as `(tenant_id, canonical_url, content_hash)` so the same accepted revision cannot create duplicate chunks or alerts. Quarantine parse failures; do not silently treat an empty extraction as a page deletion. Page removal deserves an explicit tombstone event with its own authorization and retention handling.

Promote a tenant out of the shared collection when evidence crosses a declared boundary: a contractual isolation requirement changes, its schema or embedding model diverges, its retention schedule conflicts with the shared lifecycle, or capacity tests show that its workload prevents the shared index from meeting objectives. Record the reason and make routing data-driven. The application should resolve a tenant to a collection, while the retrieval interface still requires `tenant_id`; retaining both controls makes migrations less risky.

The decision is reversible if page versions and metadata live in a source-of-truth store outside the vector index. Rebuild a tenant into a dedicated collection, compare retrieval results against the evaluation set, switch routing, and remove the old copies according to the retention policy. An index should be reproducible state, not the only record that an observed marketplace page changed.

A shared collection is therefore a starting layout, not a forever promise. The durable design is the pipeline around it: authorize scope, reject unchanged content, preserve evidence, evaluate leakage independently, and split only when measured requirements demand it.

## Sources

- https://arxiv.org/abs/2005.11401
- https://www.rfc-editor.org/rfc/rfc9110
- https://www.rfc-editor.org/rfc/rfc9111
- https://docs.python.org/3/library/html.parser.html
- https://docs.python.org/3/library/urllib.parse.html
