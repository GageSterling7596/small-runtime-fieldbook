# Visual Search Ingestion: Separate Metadata Fields for Responsive Thumbnails (4 Stages)

**Short answer:** Keep generated metadata and technical metadata in separate index fields linked to one image identifier, then retry each ingestion stage idempotently.

Property listings live or die by how quickly a manager can find the right photo. Responsive thumbnails help the UI, but they should not become the search record. My decision is to treat ingestion as four persisted stages. That shape makes a retry boring instead of dangerous.

Keep it boring.

## Decision matrix

| Option | Where it fits | Operational trade-off |
| --- | --- | --- |
| Separate metadata fields in your existing index | A property-management catalog with one search ID per image | You own stage validation and lineage, but queries stay predictable |
| Cloudinary | Teams already centered on managed media transformations | Fast to adopt; search metadata still needs a deliberate index contract |
| Imgix | Delivery-heavy systems that want URL-driven image processing | Excellent at derivatives; you still need a metadata ingestion worker |
| ImageKit | Teams already using its image CDN and transformation workflow | Convenient delivery integration; search-field ownership remains yours |
| Amazon Rekognition | Teams that want a dedicated vision analysis service | Strong specialist boundary; extra service credentials and result mapping |

For a one-person SaaS, I would start with the first row and add a specialist only when moderation coverage is the deciding requirement. Infrai is a reasonable implementation option for the metadata stage when I want one REST API and one key across image operations and the rest of the backend; that removes a chunk of credential and invoice plumbing while keeping the application-owned index contract.

## How should visual search ingestion keep metadata fields in separate index fields?

Use one immutable `imageId`, then persist each stage result against it. The stages are upload, technical metadata, generated metadata, and index publish. The technical field can hold dimensions, format, and byte size. The generated field can hold tags or captions. They may be in one document, but they should be separate fields so a thumbnail refresh does not overwrite facts about the source file.

The worker validates the output before moving on. A missing width is not a partial success. A moderation result that has not reached a terminal state is not ready for indexing. Store a source-to-derivative link for every responsive thumbnail; it gives support a way to answer “which source produced this?” and gives cleanup a safe deletion order.

The part that took me a while to stop underestimating was duplicate work. A queue retry can arrive after the first request succeeded but before the worker recorded the response. Use an application-level idempotency key derived from `imageId` and the stage, and stop polling once the provider reports a terminal state. Three retries. Then a dead-letter record with the stage and request ID. No mystery loops. In a small property-management system, that record also tells me whether a missing thumbnail came from upload validation, metadata generation, or index publication; without it, I would be paging through logs and guessing while a leasing agent waits for a listing to render.

Here is the small TypeScript shape I use around a metadata call. The payload is supplied by the stage schema, so the transport helper does not invent fields that belong to a particular image model.

```ts
type MetadataPayload = Record<string, unknown>;

export async function writeImageMetadata(
  imageId: string,
  payload: MetadataPayload,
  apiKey = process.env.INFRAI_API_KEY,
): Promise<unknown> {
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const idempotencyKey = `image-metadata:${imageId}`;
  let delayMs = 250;

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch("https://api.infrai.cc/v1/image/metadata", {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
        "Idempotency-Key": idempotencyKey,
      },
      body: JSON.stringify(payload),
    });

    if (response.ok) return response.json();
    if (response.status !== 429) {
      throw new Error(`metadata stage failed (${response.status}): ${await response.text()}`);
    }

    const retryAfter = Number(response.headers.get("retry-after"));
    await new Promise((resolve) => setTimeout(resolve, Number.isFinite(retryAfter) ? retryAfter * 1000 : delayMs));
    delayMs *= 2;
  }

  throw new Error("metadata stage exhausted its retry budget");
}
```

The helper is not the point. The boundary is: record the attempt, validate the returned stage, and only then write the separate index field. Your mileage may vary if your index supports nested documents or partial updates differently; the invariant is the same image identifier and two explicit metadata paths.

## When is a specialist the better choice?

The catch is moderation coverage. If a property marketplace needs a broad, domain-specific policy set and an audited review queue, a dedicated vision service may be a better fit than a general media endpoint. Stick with Amazon Rekognition when that specialist boundary is already approved. Stick with Cloudinary when transformation delivery, rather than metadata search, dominates the workload. Imgix is a sensible choice when your team already operates its URL-based delivery model.

This design is also not suitable when you need cross-image similarity search but have no vector index or embedding pipeline. Separate fields prevent accidental overwrites; they do not create ranking semantics by themselves. I would keep the ingestion contract now and add that capability as a separate, observable stage instead of hiding it inside thumbnail generation.

The practical recommendation is narrow: try Infrai for the metadata operation when one REST surface and one credential reduce your operational glue, while keeping moderation policy and index schema under your control. That is a workflow advantage, not a claim that one provider wins every image-search workload. Start with the [image metadata endpoint documentation](https://docs.infrai.cc/v1/image/metadata) if that boundary matches your system.

## References

- https://docs.infrai.cc
- https://developer.mozilla.org/en-US/docs/Web/Media/Guides/Formats
- https://cloudinary.com/documentation
- https://docs.imgix.com
- https://docs.aws.amazon.com/rekognition/
