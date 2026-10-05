# Logistics FAQ Retrieval: Choosing an Embedding and Vector Store with One Key

TL;DR: Choose the embedding and vector-store pair only after defining one testable contract for chunk identity, metadata filters, and replacement. For a logistics onboarding FAQ bot fed by several listing sources, the decisive behavior is whether an updated lane, cutoff time, or handling rule replaces stale chunks without leaving both versions retrievable. A single API key can shorten setup, but it cannot repair ambiguous source authority or weak freshness logic. Start with a small retrieval evaluation, then select any pair that passes it.

The useful unit of comparison is not a brand combination. It is the full path from a changed listing to the evidence returned for a question. Retrieval-augmented generation separates generation from retrieved external memory. That makes the retrieval boundary a first-class part of answer quality, not background plumbing.

## What should one embedding plus vector store combo guarantee?

An onboarding bot may receive a carrier feed, a warehouse export, and an internal operations page that describe the same service. Those records do not necessarily change together. If the carrier feed moves a pickup cutoff from 16:00 to 15:30, search must stop surfacing the older chunk once the authoritative update is indexed. Otherwise, the generator sees contradictory evidence and a fluent answer can still be wrong.

I would write the contract before opening a provider comparison page. Each chunk needs a stable logical identity, a source revision, an observed timestamp, an effective timestamp when the source supplies one, and an authority class. Upserts must be idempotent. Replacement must be explicit. Query filters must be applied during retrieval, rather than after stale candidates have already consumed the result set.

**The best pair is the one that preserves those semantics and passes your own questions.** Dimensionality, distance metric, batching limits, and filter syntax matter, but none provides a universal winner. The embedding model and index must also agree on vector dimension and similarity interpretation. Treat that agreement as a schema constraint.

Freshness wins.

## Build the smallest fresh-listing path

The following runnable example replaces the hosted embedding call and vector database with deterministic local functions. It tests the contract without pretending that a toy hash vector is a production embedding. Later, adapters can replace `embed` and `MemoryIndex` while the record shape, replacement behavior, and assertions stay fixed. That is a clean notebook-to-prod boundary.

```python
from __future__ import annotations

from dataclasses import dataclass
from datetime import datetime, timezone
import hashlib
import math
import re


@dataclass(frozen=True)
class Chunk:
    logical_id: str
    revision: int
    source: str
    observed_at: datetime
    text: str


def embed(text: str, dimensions: int = 32) -> list[float]:
    """Deterministic test vector, not a production embedding model."""
    vector = [0.0] * dimensions
    for token in re.findall(r"[a-z0-9:]+", text.lower()):
        digest = hashlib.sha256(token.encode("utf-8")).digest()
        slot = int.from_bytes(digest[:4], "big") % dimensions
        vector[slot] += 1.0
    norm = math.sqrt(sum(value * value for value in vector)) or 1.0
    return [value / norm for value in vector]


def cosine(left: list[float], right: list[float]) -> float:
    return sum(a * b for a, b in zip(left, right, strict=True))


class MemoryIndex:
    def __init__(self) -> None:
        self._rows: dict[str, tuple[Chunk, list[float]]] = {}

    def replace(self, chunk: Chunk) -> None:
        current = self._rows.get(chunk.logical_id)
        if current is None or chunk.revision >= current[0].revision:
            self._rows[chunk.logical_id] = (chunk, embed(chunk.text))

    def search(self, query: str, limit: int = 2) -> list[Chunk]:
        query_vector = embed(query)
        ranked = sorted(
            self._rows.values(),
            key=lambda row: cosine(query_vector, row[1]),
            reverse=True,
        )
        return [chunk for chunk, _ in ranked[:limit]]


index = MemoryIndex()
now = datetime.now(timezone.utc)
index.replace(Chunk(
    logical_id="lane:sha-pvg:pickup-cutoff",
    revision=41,
    source="carrier-feed",
    observed_at=now,
    text="SHA to PVG pickup cutoff is 16:00 local time.",
))
index.replace(Chunk(
    logical_id="lane:sha-pvg:pickup-cutoff",
    revision=42,
    source="carrier-feed",
    observed_at=now,
    text="SHA to PVG pickup cutoff is 15:30 local time.",
))

results = index.search("What is the SHA to PVG pickup cutoff?")
assert len(results) == 1
assert results[0].revision == 42
assert "15:30" in results[0].text
print(results[0].text)
```

One detail carries most of the weight: `logical_id` names the fact, not the ingestion event. If every crawl produces a new identifier, an upsert becomes an append and old answers remain candidates. Revision comparison also makes replaying the same batch harmless. In a real adapter, reject a vector with the wrong dimension before sending it, and persist the embedding-model identifier beside the index version so migrations cannot silently mix vector spaces.

The example has a deliberate limitation: its 32-dimensional hash vectors are suitable for contract tests, not semantic search. They make replacement behavior repeatable, yet they cannot establish retrieval quality for paraphrased onboarding questions. A hosted all-in-one path also has a clear trade-off. One key reduces credential setup, while a split design can give a team separate control over embedding rollout and index operations. Neither boundary is automatically better. A team that must keep listing data inside a specific network may find a remotely managed path unsuitable; a small team without index operators may find a self-managed store equally unsuitable. The evaluation corpus and deployment constraints resolve that choice.

That trade-off is real.

Keep chunks narrow enough that one update replaces one fact-bearing unit. A whole carrier handbook is too coarse because a cutoff change forces a broad re-embedding and may retrieve unrelated rules. A sentence fragment is too fine because qualifiers such as service level, origin, destination, and local-time basis can fall away. For listings, a practical boundary is one independently replaceable offer or policy plus the fields required to interpret it. That is a semantic rule, not a fixed token count.

## How do you compare candidates without ranking vendors?

Use the same corpus snapshot and question set for every adapter. Include ordinary onboarding questions, near-duplicate lanes, a renamed service, a deleted listing, and an update that contradicts yesterday's value. Record the expected logical IDs before measuring generated prose. Retrieval can then be scored independently from the answer model.

| Check | Passing behavior | Why it matters |
|---|---|---|
| Replacement | A newer revision leaves one retrievable logical record | Prevents stale duplicates |
| Filtering | Source, region, and active status constrain candidate search | Preserves relevant result slots |
| Deletion | A withdrawn listing disappears from retrieval | Avoids obsolete offers |
| Model migration | Old and new vector spaces use separate index versions | Prevents incomparable vectors |
| Replay | Re-ingesting one revision does not multiply records | Makes recovery predictable |
| Traceability | Returned chunks retain source and revision metadata | Supports answer inspection |

Then run two evaluations. First, calculate retrieval recall at a fixed `k` from the expected logical IDs. Second, inspect whether the final answer is supported by the returned text. Keep latency and token usage alongside those results, because larger chunks may improve context continuity while also sending irrelevant text to the answer model. This is where prompt-cost awareness belongs: in measured context, not in a guess based on a provider's onboarding screen.

Do not tune against five friendly questions. Small suites are useful for wiring, but the selection decision needs adversarial freshness cases. One revealing case indexes revision 42, replays revision 41 afterward, and verifies that 41 cannot win. Another deletes a service and asks for it by its former name.

Test the reversal.

## Chunking and freshness are one decision

Chunking is often treated as preprocessing while freshness is assigned to ingestion. In listing aggregation they meet at the replacement key. If one chunk contains three offers with different update schedules, replacing the changed offer means either rebuilding all three or retaining stale text. If three chunks omit their shared route and time-zone context, retrieval may find the right number without enough evidence to explain it.

There is no context-free chunk size to copy. Segment by update ownership first, then test whether each segment is self-interpreting. Preserve structured fields as metadata and render a concise textual form for embedding. Keep the original record available for citation or answer assembly rather than forcing every field into the embedded text.

Freshness has two clocks. `observed_at` tells the system when ingestion saw a record; an effective time tells users when a policy applies. They must not be substituted for each other. If a source does not publish an effective time, the system should not invent one.

Two clocks, two meanings.

Short-lived overlap during an index migration needs an explicit read policy. Either queries stay on the old complete index until the new one is complete, or they move through a controlled version switch. Mixing partial generations makes evaluation results hard to interpret and can expose two revisions of the same fact.

## Operate the contract after launch

Production readiness is a loop, not a one-key milestone. Log the query, selected logical IDs, revisions, source classes, index version, and identifiers of the embedding and answer models. Avoid logging sensitive listing fields unless the system's data policy permits it. These traces let a failed answer be separated into ingestion, retrieval, and generation failures.

Watch freshness lag by source and count replacement rejections caused by out-of-order revisions. Alert when an expected feed stops advancing, but let the source's normal publishing cadence define the threshold. A warehouse export updated daily and a carrier feed updated frequently should not share a guessed deadline.

Deploy adapter changes behind the same evaluation harness used for selection. Rebuild into a new index version, run the frozen questions, compare retrieval results, and switch reads only after the candidate is complete. Keep rollback at the index-version boundary. This makes a model change observable and reversible without teaching application code about a particular service.

Before calling the bot ready, read its operational story end to end: a source correction arrives; ingestion derives the same logical ID; a higher revision replaces the prior chunk; filtered search returns that revision; the answer cites its source record; and the trace shows exactly which evidence was used. Also read the failure story. A malformed record is quarantined, a late older revision is rejected, and the last valid index remains queryable.

The selection follows from that walkthrough. Pick the embedding and storage adapters that satisfy the contract, fit the team's deployment constraints, and produce the strongest retrieval results under the measured latency and context budget. The API key is an implementation detail. Fresh evidence is the product behavior.

## Sources

- https://arxiv.org/abs/2005.11401
