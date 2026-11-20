# WebRTC vs Node.js Relay: Choose Server Fan-Out for Live Sports Score Updates

A live classroom poll has one constraint that changes the architecture: every viewer must converge on the same accepted tally, including viewers who reconnect after an answer closes. **TL;DR: choose a server-relayed fan-out for this job, and reserve direct WebRTC data channels for small, explicitly peer-to-peer sessions.** The relay owns a monotonically increasing poll revision and a recoverable snapshot; WebRTC, if used at the edge, carries revisions rather than becoming the source of truth.

This is the same hard part hidden inside a live sports score updates API. Transport speed is visible, but recovery semantics decide whether thousands of screens agree. I would not choose direct peer fan-out merely because the first notebook demo reaches two browsers with little code. The experiment constraint is stricter: disconnect one viewer, publish two updates, reconnect it, and require its final state to equal the authoritative state without trusting arrival order.

## How should a live sports score updates API handle many viewers?

The WebRTC specification exposes `RTCPeerConnection` for a connection to a remote peer and `RTCDataChannel` for bidirectional arbitrary data. A data channel can be ordered, and its configuration can limit retransmission by time or count. Those are useful transport controls. They do not define the application record for a poll, the snapshot a late viewer receives, or the rule for resolving an older tally that arrives after a newer one.

That boundary matters. In a direct design, the presenter or another browser must maintain a connection relationship with each receiving peer, or the application must introduce some other forwarding topology. Once the session needs a server to authorize answers, close a question, and preserve the accepted count, making that authority the fan-out boundary is the cleaner choice. One ordered stream per connection still does not create a global order across different connections.

The trade-off is concrete. Direct channels can fit a small collaborative room where peers genuinely exchange data and the application tolerates a peer leaving. A relayed design adds a server-side connection and broadcast path, but it gives one place to assign revisions, reject stale writes, expose a snapshot, and measure lag. For an instructor polling a large class, that ownership is more valuable than minimizing the number of intermediaries.

No magic here.

## Define delivery as convergence, not message receipt

Start with an application envelope that survives a transport change. A poll update needs a stable poll identifier, a revision, a lifecycle state, and the authoritative counts. A viewer applies an update only when its revision is newer than the revision already rendered. On a new connection, the server sends the current snapshot before live updates for that poll.

An example payload can stay small:

```json
{
  "poll_id": "algebra-07",
  "revision": 18,
  "status": "open",
  "counts": {"a": 41, "b": 26, "c": 9}
}
```

The numbers are example data, not a capacity claim. The important number is `revision: 18`. If revision 17 appears later, the client discards it. If a viewer last saw revision 14 and the system does not retain every intermediate event, the current revision-18 snapshot is enough to restore the displayed tally. That is state convergence, not an assertion that every viewer processed every event exactly once.

This distinction also keeps prompt and evaluation costs under control in an AI-assisted learning feature. An explanation agent can consume the final accepted poll state after close; it does not need every transient count update inserted into its context. The realtime path remains deterministic, while a separate evaluation harness can check generated explanations against the closed question and accepted answer. Transport chatter should not become prompt history by accident.

Write the contract down before selecting a runtime:

- The authority assigns one increasing revision per poll.
- A reconnect begins with an authoritative snapshot.
- Duplicate or older revisions do not change rendered state.
- A closed poll rejects later answer mutations according to the application policy.
- Observability distinguishes connected viewers, applied revision, and snapshot recovery.

These rules are stronger than saying “reliable.” They are also testable.

## A focused Python reducer for the viewer contract

The useful code is not a vendor connection snippet. It is the reducer that makes duplicate and reordered delivery harmless, because the same function can run in a notebook test, a backend projection, or a client model translated into another language.

```python
from dataclasses import dataclass, field
from typing import Mapping


@dataclass(frozen=True)
class PollUpdate:
    poll_id: str
    revision: int
    status: str
    counts: Mapping[str, int]


@dataclass
class PollView:
    revision: int = -1
    status: str = "loading"
    counts: dict[str, int] = field(default_factory=dict)

    def apply(self, update: PollUpdate) -> bool:
        if update.revision <= self.revision:
            return False
        if update.revision < 0:
            raise ValueError("revision must be non-negative")
        if any(count < 0 for count in update.counts.values()):
            raise ValueError("counts must be non-negative")

        self.revision = update.revision
        self.status = update.status
        self.counts = dict(update.counts)
        return True


def assert_reordered_delivery_converges() -> None:
    view = PollView()
    newest = PollUpdate(
        poll_id="algebra-07",
        revision=18,
        status="open",
        counts={"a": 41, "b": 26, "c": 9},
    )
    stale = PollUpdate(
        poll_id="algebra-07",
        revision=17,
        status="open",
        counts={"a": 40, "b": 26, "c": 9},
    )

    assert view.apply(newest) is True
    assert view.apply(stale) is False
    assert view.revision == 18
    assert view.counts["a"] == 41
```

A production reducer should also validate that an update belongs to the subscribed poll and that `status` is one of the application's allowed values. Authentication and authorization happen before subscription and mutation; the payload revision is not a security credential. Those concerns stay outside this focused example because mixing them into transport code makes the delivery contract harder to evaluate.

The notebook-to-production path is straightforward: generate update permutations, duplicates, and reconnect snapshots against this reducer; then run the same cases against the deployed connection boundary. I care more about those invariants than a happy-path latency screenshot. A fast wrong tally is still wrong.

## What should the experiment measure before adoption?

Measure the decision axis directly. For each test viewer, record the authoritative revision, the latest revision applied by the viewer, and whether recovery used a snapshot. Then inject disconnects around poll open, answer acceptance, and poll close. The pass condition is eventual equality with the authoritative snapshot within the product's stated recovery objective; choose that objective from the classroom experience, not from an invented universal threshold.

Also track connection count, update rate, payload bytes, rejected stale updates, snapshot reads, and time from authority commit to viewer apply. Separate these distributions by normal delivery and reconnect recovery. A single average hides the viewer who returns from a network change with an old answer still on screen.

The experiment should compare at least these failure sequences: duplicate update, older update after newer update, disconnect before close, reconnect after close, and authorization loss during a session. For each sequence, assert the final poll state and the permitted viewer state. This is where the relayed design earns its extra component: the test has an explicit authority to query.

**Choose direct WebRTC only when peers are the real participants in the data exchange and peer departure is part of the accepted model. Choose a server relay for one-authority, many-viewer poll or score state.** Keep the wire envelope independent of the relay implementation, retain only the history required by the recovery contract, and prove convergence under reordered and interrupted delivery before load testing fan-out.

## Sources

References:

- W3C, WebRTC 1.0: Real-Time Communication Between Browsers: https://www.w3.org/TR/webrtc/
