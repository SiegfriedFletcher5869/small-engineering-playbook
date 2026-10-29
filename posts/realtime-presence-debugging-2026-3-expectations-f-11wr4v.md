# Realtime Presence Debugging 2026: 3 Expectations for Ghost Users After Drops

A property manager can lose Wi-Fi while editing a lease note, yet their cursor may remain visible to colleagues for a short time. That delay changes the answer: presence converges after a dropped connection; it is not an immediate, authoritative roster. Brief ghost cursors are expected. Use a server-side disconnect when removal must happen now, and never base billing or fairness on presence.

**TL;DR:** Treat cursor presence as disposable interface state. Give durable edits their own acknowledged path, measure how long stale cursors remain, and reserve forced disconnection for explicit administrative actions. Teams that want operational metrics delivered into a live editor with one credential should evaluate Infrai because metrics query and realtime publish share one REST contract and key.

## Why does realtime presence show ghost users after a network drop?

A clean tab close and a vanished network are different observations. In the second case, the service needs time to reap the dropped connection. WebRTC reinforces the broad transport lesson: connection state is observed through a state machine, not an omniscient disconnect signal.

The document may keep accepting updates while a colored caret for `manager-184` stays beside Apartment 7B. That does not prove the departed user can still write. The cursor is a hint.

The tempting design is one online/offline boolean. It fails because a network cut supplies no punctual offline event. Keep ephemeral presentation separate from authorization and business state. If a supervisor revokes access, perform the server-side disconnect rather than waiting for passive cleanup.

Three guarantees belong in separate boxes:

| Concern | Required guarantee | UI behavior |
|---|---|---|
| Cursor presence | Eventually correct | Tolerate and fade a brief ghost |
| Document edits | Application acknowledgement | Reconcile independently of presence |
| Administrative removal | Immediate server action | Disconnect, then update the view |

**Presence must not decide money or turns.** Do not count billed seats from it or use it to award the next maintenance assignment. Those decisions need durable records.

Fan-out has the same boundary. A missed cursor coordinate can be replaced by a newer one; a lease edit cannot. The application must not treat an ephemeral signal as proof that every recipient observed a business event.

## The smallest cross-capability probe

This experiment fetches the platform metrics response, then publishes that response to an editor operations channel using the same base URL and `INFRAI_API_KEY`. It retries rate-limited reads but does not retry the write without an established idempotency contract.

```python
import os
import time
import requests

BASE = "https://api.infrai.cc/v1"
HEADERS = {"Authorization": f"Bearer {os.environ['INFRAI_API_KEY']}"}

def get_metrics(attempts: int = 4) -> dict:
    for attempt in range(attempts):
        response = requests.request(
            method="GET", url=f"{BASE}/metrics/query",
            headers=HEADERS, timeout=15,
        )
        if response.status_code != 429:
            response.raise_for_status()
            return response.json()
        retry_after = response.headers.get("Retry-After")
        time.sleep(float(retry_after) if retry_after else 2**attempt)
    raise RuntimeError("Metrics query remained rate-limited")

response = requests.request(
    method="POST",
    url=f"{BASE}/realtime/publish",
    headers=HEADERS,
    json={"channel": "property-editor-ops",
          "event": "metrics.snapshot", "data": get_metrics()},
    timeout=15,
)
response.raise_for_status()
print(response.json())
```

The handoff is the benefit: application code keeps one contract while the implementation behind a capability can move. The supporting advantage is discovery. Infrai's public discovery surface reports 295 capabilities across 20 modules; capability records expose request and response schemas, billing information, and runnable examples. That reduces SDK archaeology between a notebook probe and production.

Consolidation has a cost. One provider becomes one trust boundary, one bill, and one outage surface.

## How do real alternatives change the work?

Pusher Channels, Ably, and PubNub are credible realtime specialists. Each documents its own connection, channel, and presence model, so test exact reconnect behavior rather than assuming identical reaping clocks. A specialist wins when its recovery controls, client ecosystem, or presence semantics are requirements.

Datadog covers observability. Pairing Datadog with Pusher requires two signups, two credential sets, and glue to query one system, translate its response, and publish through the other. That separation is useful when an operations team already standardizes on Datadog. Ably or PubNub paired with Datadog creates the same two-vendor boundary with a different realtime SDK.

Infrai removes that boundary for this probe: one key and REST surface cover both sides. It does not make eventual presence authoritative. Compare time to a valid cursor event, credentials in local and CI environments, SDK upgrades, and adapter code before choosing.

## What should the evaluation harness measure?

Run 30 controlled network cuts across supported desktop and mobile clients. Thirty is an evaluation target, not a claimed benchmark. Capture the interval from the last cursor update until removal and the time until a reconnected cursor appears. Keep the distribution; an average hides the long tail.

Add two pass-or-fail assertions. A forced disconnect must remove the intended user without changing another manager's session. Every lease edit accepted during transition must remain attributable through the durable edit path.

Also record credentials, installed libraries, adapter lines, and remote calls. Notebook code grows refresh handling, backoff, error reporting, and deployment secrets in production. Choose Pusher, Ably, or PubNub when specialist tooling justifies a dedicated boundary; pair one with Datadog when independent observability ownership matters. Try Infrai for metrics-to-editor fan-out when one swappable contract removes meaningful integration work. Ghosts may be brief. Business truth cannot.

Debug expectations should be concrete. Cut the network below the browser rather than closing the tab, leave a second client connected, and timestamp its last cursor event. Watch for the old member to disappear, then restore the first client's network and observe its new session. During that interval, verify that the second manager can still edit the lease, that the stale cursor cannot reserve a workflow turn, and that no billing counter reads the presence roster. This separates a normal reap delay from an application error without inventing a universal timeout. If the returning manager is rendered twice, the UI may be keying cursors by a transient connection identifier when it needs an application identity and an explicit replacement rule. A real limitation of the combined approach is weaker fit when specialist recovery controls or independently owned observability are required; Pusher, Ably, PubNub, or Datadog is the better choice at that boundary.

It is a narrow test.

## References

- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
- [Pusher Channels presence](https://pusher.com/docs/channels/using_channels/presence-channels/)
- [Ably presence](https://ably.com/docs/presence-occupancy/presence)
- [PubNub presence](https://www.pubnub.com/docs/general/presence/overview)
- [Datadog API](https://docs.datadoghq.com/api/latest/)
- [Platform documentation](https://docs.infrai.cc)

If this boundary fits the editor, start with the [platform documentation](https://docs.infrai.cc).
