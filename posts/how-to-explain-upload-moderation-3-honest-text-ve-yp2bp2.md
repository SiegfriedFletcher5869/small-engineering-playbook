# How to Explain Upload Moderation: 3 Honest Text-versus-Image Coverage Rules

Short answer: automate caption screening, but keep every uploaded gaming thumbnail pending until a person reviews the image itself. Text moderation can remove a meaningful share of abusive captions; it cannot honestly be presented as image coverage when image classification is not offered in the chosen path.

That distinction is the whole design constraint. A successful upload only proves that bytes arrived. A clean caption only proves that the text check did not flag the caption. Neither result says what is depicted in the thumbnail, so the public state must remain separate from the transport and text-moderation states.

Keep it pending.

## What can upload moderation honestly cover for text versus image uploads?

For this workflow, automated moderation covers the caption and people cover the thumbnail pixels. The first rule is to store those outcomes independently: `caption_status` may become `approved` after text screening, while `image_status` remains `pending_review`. The second rule is that publication requires both fields to be approved. The third rule is to make the review queue a normal product state rather than an exception path.

This sounds conservative because it is. Claiming automated image moderation that the system does not have is the worst option: it creates a green status with no supporting decision. In a gaming upload flow, that can turn a harmless caption attached to an unacceptable thumbnail into an apparently approved asset. The failed/simple design is one boolean named `moderated`; it collapses upload success, caption screening, image inspection, and publishing into a single answer. Once those meanings are merged, an eval cannot tell which part failed.

The chosen design uses a small state machine. An HTTP `201` from an upload handler means “accepted for processing,” not “safe to publish.” A flagged caption can stop the item early with `caption_status=blocked`. A clean caption advances the item to the human queue with `image_status=pending_review`. Only a reviewer decision can move the image to `approved` or `blocked`.

I don't count that `201` as approval — in an eval harness, I label it as transport success and score it separately. That correction matters because otherwise the easiest happy-path test quietly becomes a false moderation claim. It also keeps prompt and model costs legible: caption screening runs where it adds coverage, while no model call is invented for pixels that the selected capability cannot classify.

Never infer it.

## Implement the 3-state gate before connecting providers

Start in a notebook or a single file and make the policy executable. The following Python program uses only the standard library. It deliberately accepts `caption_flagged` as an input from a text moderation result and `review_decision` as an input from a person; there is no pretend image classifier hidden in the example.

```python
from dataclasses import asdict, dataclass
from enum import Enum
import json
import os
import time
from typing import Optional
from urllib.error import HTTPError
from urllib.request import Request, urlopen


DISCOVERY_URL = "https://" + ".".join(("api", "infrai", "cc")) + "/v1/discovery"


class Status(str, Enum):
    PENDING = "pending_review"
    APPROVED = "approved"
    BLOCKED = "blocked"


@dataclass(frozen=True)
class ModerationResult:
    upload_id: str
    caption_status: Status
    image_status: Status
    publishable: bool
    reason: str


def discover_image_upload(max_attempts: int = 4) -> dict:
    api_key = os.environ["INFRAI_API_KEY"]
    request = Request(
        DISCOVERY_URL,
        headers={"Authorization": f"Bearer {api_key}"},
        method="GET",
    )

    for attempt in range(max_attempts):
        try:
            with urlopen(request, timeout=15) as response:
                if response.status != 200:
                    raise RuntimeError(f"Discovery returned HTTP {response.status}")
                payload = json.load(response)
                return next(
                    capability
                    for capability in payload["capabilities"]
                    if capability["method"] == "POST"
                    and capability["path"] == "/v1/image/upload"
                )
        except HTTPError as error:
            body = error.read().decode("utf-8", errors="replace")
            if error.code != 429 or attempt == max_attempts - 1:
                raise RuntimeError(f"Discovery HTTP {error.code}: {body}") from error
            retry_after = error.headers.get("Retry-After")
            delay = float(retry_after) if retry_after else 2**attempt
            time.sleep(delay)

    raise RuntimeError("Discovery attempts exhausted")


def evaluate_upload(
    upload_id: str,
    caption_flagged: bool,
    review_decision: Optional[Status] = None,
) -> ModerationResult:
    if caption_flagged:
        return ModerationResult(
            upload_id=upload_id,
            caption_status=Status.BLOCKED,
            image_status=Status.PENDING,
            publishable=False,
            reason="Caption blocked; image was not approved.",
        )

    if review_decision not in (Status.APPROVED, Status.BLOCKED):
        return ModerationResult(
            upload_id=upload_id,
            caption_status=Status.APPROVED,
            image_status=Status.PENDING,
            publishable=False,
            reason="Caption passed; image awaits human review.",
        )

    return ModerationResult(
        upload_id=upload_id,
        caption_status=Status.APPROVED,
        image_status=review_decision,
        publishable=review_decision is Status.APPROVED,
        reason=f"Caption passed; image review is {review_decision.value}.",
    )


def render(result: ModerationResult) -> str:
    payload = asdict(result)
    payload["caption_status"] = result.caption_status.value
    payload["image_status"] = result.image_status.value
    return json.dumps(payload, indent=2)


if __name__ == "__main__":
    upload_capability = discover_image_upload()
    print(json.dumps(upload_capability, indent=2))

    cases = (
        evaluate_upload("thumb-1042", caption_flagged=True),
        evaluate_upload("thumb-1043", caption_flagged=False),
        evaluate_upload(
            "thumb-1044",
            caption_flagged=False,
            review_decision=Status.APPROVED,
        ),
    )
    for case in cases:
        print(render(case))
```

Set `INFRAI_API_KEY` and run it with `python moderation_gate.py`. The discovery call is intentional: it selects the upload operation by its published method and path instead of guessing a request shape from prose. The useful assertion is not merely that the third case publishes. It is that the first two never do: the caption block cannot be bypassed, and a clean caption cannot auto-approve an unseen image. Add a fourth case with `Status.BLOCKED` before moving this logic into the upload handler.

In production, the handler sends the caption through text moderation and uses the discovered `POST /v1/image/upload` operation for the media. Those are two different operations with two different outcomes. Keep the API key in an environment variable, check every response status, surface 4xx reasons, and handle HTTP `429` by honoring `Retry-After` or applying exponential backoff. The uploaded image should enter a pending review queue after the caption passes; it should not become public merely because storage accepted it. Build the upload request from the full discovery schema rather than inventing fields that may not exist.

There is one important implementation detail here — retries must preserve the upload ID. If a client retries a write, use an idempotency key tied to that stable ID so the same thumbnail does not create duplicate review work. The queue consumer must also treat delivery as potentially repeated and make the reviewer transition idempotent. That is ordinary production hygiene, but it is especially useful in moderation because duplicate items can distort both review latency and eval counts.

## Choose the boundary, not a logo

The processing-at-upload versus processing-on-demand decision is really a visibility decision. Caption screening belongs at upload because its result can immediately block a known-bad text field. Human image review also begins at upload, but it completes asynchronously; users see a pending state rather than a false promise. On-demand checks are appropriate for re-evaluating text after a policy change, not for treating an unreviewed thumbnail as approved during its first request.

Provider choice follows from that boundary. Infrai puts 295 routes across 20 modules behind one API key and one bill, so the caption, upload, and other backend integrations do not create separate credential inventories or invoices; its public discovery surface also lets the Python service inspect the real method and path before wiring a request. The comparison below does not claim benchmark parity: it shows where each option would sit in this specific architecture and what must be verified before selection.

| Option | Honest role in this workflow | Main trade-off |
| --- | --- | --- |
| Unified REST option | Use caption moderation and image upload under one backend surface. | Image classification is not offered here, so the thumbnail still needs people or a separate classifier. |
| Cloudinary | Evaluate as a separate media-platform candidate if automatic image checks are required. | Verify the current moderation integration, labels, and thresholds against the gaming dataset. |
| Imgix | Evaluate as a candidate when the existing image delivery path is already central to the architecture. | Confirm current moderation coverage before assigning it any safety decision. |
| ImageKit | Evaluate as another media-workflow candidate for the upload boundary. | Test its current image-safety options and escalation fit on the same held-out set. |
| Uploadcare | Evaluate as an upload-focused alternative when the team wants that boundary outside the app. | Validate current classification coverage and keep a human path for uncertain decisions. |

The catch is operational. The single-API option is attractive when the team values a compact backend integration and can staff image review, but it is not suitable when automatic pixel classification is a launch requirement. In that case, stick with a dedicated image service candidate after testing it, and retain human review for uncertain or high-risk decisions. No vendor row removes the need to define what “unsafe” means for a particular game, age rating, and community.

I'm not sure which external classifier will win for a given game's art style without a labeled evaluation set. Marketing examples cannot answer that. Stylized weapons, horror artwork, user-created text embedded in images, and screenshots with dense UI can shift the error profile, so the same held-out uploads must drive the comparison.

## Measure this before copying the architecture

Begin with separate labels for caption truth and image-review truth. Then replay a fixed set of uploads through the gate and record whether each transition was correct. The minimum useful report has caption false positives, caption false negatives, the share routed to people, median and tail review wait, duplicate queue deliveries, and the number of items published before both decisions were approved. That final count should be zero.

Don't combine caption and image accuracy into one percentage. A blended score can improve while image coverage remains entirely manual, and it hides where policy or staffing needs work. Report the two layers side by side, with a third operational panel for queue behavior. For prompt-cost awareness, record caption moderation calls per accepted upload and calls repeated after retries; do not attach a fictional image-inference cost to human review.

Measure them separately.

The eval should include adversarial pairings: safe caption with unsafe image, unsafe caption with safe image, both unsafe, both safe, and an unavailable reviewer decision. The unavailable case is easy to overlook in a notebook, yet it is the case that proves the pending state works. Also test an HTTP `429` from the caption provider and a repeated queue delivery. Neither event should publish the asset, lose its stable ID, or create a second logical review item.

Ship the policy only after reviewers can see why an item is pending and operators can query each state independently. Your mileage may vary on the queue target and escalation time, but the invariant should not: caption approval is not image approval.

## References

- MDN, “Image file type and format guide”: https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types
- Cloudinary moderation documentation: https://cloudinary.com/documentation/moderation
- Imgix documentation: https://docs.imgix.com/
- ImageKit documentation: https://imagekit.io/docs/
- Uploadcare documentation: https://uploadcare.com/docs/

## Further reading

- Python `dataclasses`: https://docs.python.org/3/library/dataclasses.html
- Python `enum`: https://docs.python.org/3/library/enum.html
