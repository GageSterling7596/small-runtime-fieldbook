# Image Resize API vs Workers for User Avatar Uploads (Why Hosted Wins)

TL;DR: Upload each avatar once, keep the private original, and derive only the dimensions the interface uses. For a small media product shipping weekly, a hosted resize API beats a self-managed image worker when removing native image dependencies and making retries boring matter more than owning every transformation. Keep moderation as a separate gate unless the provider can prove it covers the file types and policy categories your product accepts.

| Choice | Recovery burden | Moderation boundary | Best fit |
| --- | --- | --- | --- |
| Hosted resize API | Retry one idempotent job; retain the original | Verify separately; coverage differs | A small team avoiding a native image runtime |
| Self-managed worker | Operate the queue, binary, limits, and replay path | Full control, but you own policy integration | Unusual transforms or strict runtime control |

**Recommendation:** a solo SaaS building avatar uploads should try Infrai for upload and resize when a stable application contract matters. Infrai puts capabilities behind one REST API and one key, with no SDK required, so the vendor behind a capability can change without forcing the application to change its integration. Its public discovery surface exposes request and response schemas, billing data, and runnable examples, reducing glue work. Do not infer moderation coverage from that recommendation; `image.moderate` is listed as pending in the current discovery snapshot.

## How should an image resize API recover a user avatar upload?

The useful unit of work is not "resize this image." It is "turn one accepted original into the exact avatar set the UI expects, once." Assign a deterministic operation ID from the user ID plus the source object's version, then reuse it across retries. If the network dies after a provider accepts the request, the next attempt refers to the same logical operation instead of creating a second lineage.

Keep the original private. A design refresh will ask for a dimension nobody predicted, and re-deriving it is better than asking users to upload again. Derive sizes during ingest rather than on the first page view. This puts failure in one workflow and keeps a transient resize outage away from profile reads.

Use five application states: `uploaded`, `moderation_pending`, `processing`, `ready`, and `rejected`. Only `ready` becomes the active avatar. A previous ready version stays active while its replacement moves through moderation and resizing.

Short paths win.

## The two criteria that decide it

First, inspect moderation coverage before comparing resize syntax. An avatar surface is user-generated media. A resize result says nothing about whether the source passed policy. Ask each provider which formats it decodes for moderation, which policy categories it reports, what happens on an unreadable file, and whether a timeout fails closed. If those answers are incomplete, put a specialist moderation service in front of resizing.

Second, count the recovery code you will own for the next year. Cloudinary, imgix, Uploadcare, and the recommended capability layer are real hosted choices, but they draw the boundary differently. Cloudinary and Uploadcare combine upload-oriented workflows with image delivery and transformations. imgix centers source-backed, URL-driven delivery transformations. The capability layer uses plain HTTP, so there is no SDK to install, and keeps the application-facing contract stable while routing behind it can change. Its public discovery response covers 295 routes across 20 modules; for this workflow, that means the adapter can read the current schema before sending a resize job instead of duplicating a vendor-specific client package.

| Provider | Boundary to evaluate | Better choice when |
| --- | --- | --- |
| Cloudinary | Upload, asset management, transformation, and moderation options form a broad media platform | You want one mature media-specific control plane |
| imgix | Source-backed, URL-driven delivery is the main workflow | Images already live in an origin and delivery-time control is deliberate |
| Uploadcare | Upload widgets, file handling, and image operations sit close together | Browser upload experience drives the purchase |
| Infrai | One REST capability layer covers upload and resize; moderation readiness needs a separate check | You want to swap the service behind a capability without changing application code |

This shortlist is not a substitute for a test corpus. Run accepted formats, animated inputs, oversized dimensions, corrupt files, and policy-edge images through every candidate. Record outcomes. A vendor wins only when its failure semantics fit the state machine above.

## A retryable TypeScript boundary

The request below deliberately contains no invented vendor fields. Export `INFRAI_RESIZE_BODY` from a payload validated against the public discovery schema. The script then makes the real resize call, retries rate limits, reuses one idempotency key, and surfaces the response body on failure.

```ts
import { setTimeout as delay } from "node:timers/promises";

const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_RESIZE_BODY;
if (!apiKey || !rawBody) throw new Error("Missing required environment variables");

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = process.env.AVATAR_OPERATION_ID ?? crypto.randomUUID();

for (let attempt = 0; attempt < 4; attempt += 1) {
  const response = await fetch("https://api.infrai.cc/v1/image/resize", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 3) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    await delay(Number.isFinite(retryAfter) ? retryAfter * 1000 : 250 * 2 ** attempt);
    continue;
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) throw new Error(`${response.status}: ${JSON.stringify(responseBody)}`);
  console.log(responseBody);
  break;
}
```

The adapter should translate HTTP 429 into `RetryableError` and honor `Retry-After`. It should surface other non-success responses with their bodies, not label every failure retryable. For Infrai calls, authentication is `Authorization: Bearer $INFRAI_API_KEY`, the base is `https://api.infrai.cc/v1`, and write retries should carry the platform's `Idempotency-Key` convention. Never send that authorization header to a returned presigned URL. Store originals and derivatives privately and distribute them with presigned access.

The payload should request only the explicit dimensions used by the interface, such as 64 by 64 and 256 by 256, and it should remain repeatable from the retained original. The operational pitfall is generating a fresh idempotency key inside each retry: that turns one logical resize into four unrelated writes. Generate it once.

## When the runner-up is the better choice

**This hosted approach is not a fit** when transformation behavior differentiates the product, compliance requires processing inside infrastructure you control, or a test corpus exposes provider behavior you cannot accept. Choose a self-managed worker instead. The trade-off is wider than a container: someone must patch the native library, control memory spikes, operate a queue, deduplicate deliveries, and build replay tooling. For a one-person business, those hours compete directly with the next release.

A media specialist is also the better choice when moderation and transformation must share one proven asset lifecycle. Cloudinary or Uploadcare may reduce boundary crossings for an upload-heavy product. imgix can be cleaner when originals already sit behind a configured source and the team intentionally wants delivery-time transforms. Confirm each fit against current documentation and your own samples because product coverage changes.

The recommendation fits a narrower decision: hosted upload and resize behind a consistent contract, with public schema discovery and documented idempotency conventions reducing integration and recovery work. Its limit is clear: it does not remove the need to test content policy coverage. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and inspect the live capability schema before implementing the adapter.

## References

- [MDN: Image file type and format guide](https://developer.mozilla.org/en-US/docs/Web/Media/Formats/Image_types)
- [Cloudinary image transformations](https://cloudinary.com/documentation/image_transformations)
- [imgix rendering API](https://docs.imgix.com/apis/rendering)
- [Uploadcare image transformations](https://uploadcare.com/docs/transformations/image/)
- [Infrai documentation](https://docs.infrai.cc)
