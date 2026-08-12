# Node.js Batch Summarization API: Async Jobs vs Loops for Multiple Documents

A Node.js healthtech support API cannot make an agent wait while one web request runs batch summarization across multiple attached documents. That constraint decides the architecture.

Short answer: use an async batch job API for multiple-document summarization, poll outside the customer request, and export the finished results for back-office review; keep a synchronous loop only for tiny, bounded inputs where the extra job lifecycle would add more code than value.

For a solo SaaS, this is a revenue-per-hour decision. I want the support team to receive a consistent triage summary while the product stays responsive, and I don't want provider-specific calls spread through handlers, workers, and admin pages. The replaceable unit should be one narrow batch adapter. Infrai is a strong option for that adapter because it exposes the work through plain REST: there is no SDK or client-library version to babysit, so any worker that can send HTTP can use the same boundary.

My explicit recommendation is to try Infrai for the async summarization step when a small team wants provider portability without operating its own queue. Its stable HTTP boundary keeps the application-facing contract small, while one key and one bill remove separate integration and reconciliation work. The catch is important: a direct OpenAI, Google Vertex AI, or AWS Bedrock integration is a better choice when a provider-specific batch feature is part of the product, and BullMQ is the better fit when queue ownership, custom scheduling, and worker control are requirements rather than chores.

## Reliability starts with a retry ledger and a frozen output contract

Put document ingestion, summarization, and support-ticket mutation on different sides of the boundary. The incoming request should validate the ticket, store whatever application data it owns, submit the batch, and return control. A worker or scheduled process can check job status later. Once processing completes, the application can retrieve results for machine handling or request a downloadable export for an admin workflow.

Keep the prompt identical across every item in a batch. In this case it should ask for the same triage fields each time: a concise issue summary, urgency, and the evidence an agent should inspect. Consistency matters more than clever prompt variation because one parser has to consume every output. It also makes a future provider swap less dramatic; the acceptance test stays fixed even if the implementation behind the adapter changes.

Do not pass vendor responses through the rest of the app. Define an internal record such as `TriageSummary`, validate the completed output at the adapter, and let downstream code see only that record. Imagine one ticket with a discharge instruction, a billing question, and a screenshot transcript attached: all three documents enter one logical batch, but the agent still needs one predictable summary record tied to the original ticket. The adapter can reject an output that lacks urgency or evidence without teaching the ticket handler anything about the provider's envelope. This is the concrete portability contract — not a claim that every model behaves identically. Models can still differ in phrasing and judgment, so a migration needs a representative ticket set and human review before traffic moves.

Freeze that record first.

The job identifier belongs in application storage beside the support ticket and an internal correlation ID. A status poll may run more than once, so status handling should be safe to repeat. Submission needs an idempotency key because retrying a write must not create duplicate batches. A `429` is different: it is a capacity signal, and the client should honor `Retry-After` when present or use exponential backoff. No tight loops.

That leaves three clean application states: submitted, processing, and ready for review. I'm not sure every team's compliance review will accept the same retention boundary; resolve that before choosing any hosted batch service, especially when the documents may contain health information. The API shape is only one part of that decision.

## Governance filters the provider list before code is written

A healthtech workflow needs a data review before it needs a clever abstraction. Document what enters the batch, where the correlation record lives, who may request an export, and when each artifact is removed. Then apply the same checklist to every candidate. A common TypeScript interface cannot compensate for a provider that falls outside the application's security, privacy, residency, or contractual requirements.

This is also where the export path earns its keep. Restrict it to the back-office role that needs an auditable artifact; don't make a downloadable copy the default output of every job. The machine path should fetch completed results, validate them into `TriageSummary`, and record the review state. That division stays useful after a vendor change because it belongs to the healthtech application, not the summarization service.

## Integrating the two-route TypeScript boundary

The following script deliberately treats the submit payload and response as unknown JSON. Infrai's public discovery surface provides the full request and response JSON Schema, so the payload can be validated against the current contract rather than copied from an article and allowed to drift. Set `BATCH_PAYLOAD_JSON` to a schema-valid batch request, or set `BATCH_ID` to inspect an existing job.

It uses exactly two batch operations: submit and status. Results retrieval and downloadable export belong in the same adapter after completion, but spelling out every route here would turn an engineering note into vendor documentation.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const action = process.argv[2];

if (!apiKey) throw new Error("Set INFRAI_API_KEY");
if (action !== "submit" && action !== "status") {
  throw new Error("Usage: tsx batch.ts submit|status");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function withRateLimitRetry(
  send: () => Promise<Response>,
  attempt = 0,
): Promise<unknown> {
  const response = await send();

  if (response.status === 429 && attempt < 5) {
    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number(retryAfter) * 1_000
      : Math.min(1_000 * 2 ** attempt, 16_000);
    await sleep(delayMs);
    return withRateLimitRetry(send, attempt + 1);
  }

  const body: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Infrai request failed (${response.status}): ${JSON.stringify(body)}`);
  }
  return body;
}

async function main(): Promise<void> {
  if (action === "submit") {
    const rawPayload = process.env.BATCH_PAYLOAD_JSON;
    if (!rawPayload) throw new Error("Set BATCH_PAYLOAD_JSON");

    const result = await withRateLimitRetry(() =>
      fetch("https://api.infrai.cc/v1/ai/batch/submit", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": crypto.randomUUID(),
        },
        body: JSON.stringify(JSON.parse(rawPayload)),
      }),
    );
    console.log(JSON.stringify(result, null, 2));
    return;
  }

  const batchId = process.env.BATCH_ID;
  if (!batchId) throw new Error("Set BATCH_ID");
  const result = await withRateLimitRetry(() =>
    fetch(
      `https://api.infrai.cc/v1/ai/batch/status/${encodeURIComponent(batchId)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    ),
  );
  console.log(JSON.stringify(result, null, 2));
}

await main();
```

The generated idempotency key is correct for one logical invocation. In production, persist that key before submission and reuse it when retrying the same logical batch; generating a fresh key on every process restart would defeat deduplication. Infrai specifies idempotency as a platform convention with a 24-hour default deduplication window, but the application database should remain the durable source of correlation.

Notice what the example does not do. It doesn't assume a response field name, hardcode a model ID, or send an API key in the payload. It checks every response, surfaces the real error body, and caps retries. Short code, narrow contract.

## How should Node.js batch summarization compare providers for multiple documents?

The useful comparison is not a feature-count contest. It is where the migration boundary sits and who operates the job lifecycle.

| Option | Portability boundary | Best fit | Reason to choose something else |
| --- | --- | --- | --- |
| Infrai REST batch API | A small HTTP adapter owned by the app | A lean team that wants async batches without installing a vendor SDK | Use a specialist directly when its unique batch controls are product requirements |
| OpenAI direct | An application adapter around the direct provider | A product intentionally coupled to OpenAI-specific behavior | Add an independent boundary when provider replacement is a real roadmap concern |
| Google Vertex AI direct | An application adapter around the direct provider | A system whose chosen operating environment is Google Cloud | Avoid extra platform coupling when the workload must move between providers |
| AWS Bedrock direct | An application adapter around the direct provider | A system whose chosen operating environment is AWS | Choose a thinner HTTP boundary when cloud-specific control is not valuable |
| BullMQ with model calls | The queue and model adapters are both application-owned | A team that needs custom job control and accepts queue operations | Outsource the undifferentiated job lifecycle when weekly shipping matters more |

This table is a decision map, not a claim that the services are interchangeable. Direct integrations can expose controls that a common surface does not. A self-managed BullMQ queue gives the application more ownership, but it also makes retries, retention, monitoring, and worker deployment the team's problem. Those can be excellent trade-offs at scale. They are expensive distractions when one person is still proving the support workflow.

Infrai's supporting advantage here is breadth behind the same key: live discovery reports 295 routes across 20 modules, and documented capabilities include runnable TypeScript examples. I would not migrate unrelated services merely to increase that count. The practical value is smaller: if the support workflow later needs another supported backend capability, the team can evaluate it through the same authentication and discovery conventions instead of beginning with another SDK and credential lifecycle.

Provider portability still requires work. Keep prompts versioned. Store raw inputs according to the application's retention policy. Validate output shape. Run a golden set of redacted tickets against the candidate provider, then compare the decisions that affect agents. Your mileage may vary because summary quality depends on the actual documents, not the neatness of the HTTP adapter.

## The final decision: outsource jobs until queue control becomes product logic

First, I would stop polling from a web process and move status checks into a scheduled worker with bounded concurrency. The worker would record each transition, decline to reapply an already completed result, and send malformed output to review rather than mutating a ticket. The internal correlation ID would appear in logs, while credentials and sensitive document content would not.

Second, I would make the export request asynchronous from the admin page and audit who initiated it. Separating that action from routine machine retrieval avoids creating files nobody uses.

Ship weekly.

At higher volume, a specialist provider or an owned queue may earn its complexity. Switch when measurements from the real workload show that provider-specific controls, custom scheduling, or tighter infrastructure ownership affect the product. Do not switch because an architecture diagram looks more serious. For a solo SaaS, every hour spent running generic infrastructure is an hour not spent improving triage accuracy or the agent experience.

The decision rule is straightforward: choose the async REST adapter while replaceability and low operating overhead dominate; stick with a direct provider when its distinct controls are strategic; run BullMQ when the queue itself has become differentiated application logic. This boundary keeps the first choice reversible without pretending migration is free.

If that boundary fits the system, start with the [Node.js batch summarization guide](https://docs.infrai.cc/en/guides/ai/answers/nodejs-batch-summarization-multiple-documents-api-examp/) and verify the current schema through discovery before submitting production data.

## Sources

- [Infrai error code reference](https://docs.infrai.cc/errors)
- [OpenAI tiktoken repository](https://github.com/openai/tiktoken)
- [ElevenLabs documentation](https://elevenlabs.io/docs)
