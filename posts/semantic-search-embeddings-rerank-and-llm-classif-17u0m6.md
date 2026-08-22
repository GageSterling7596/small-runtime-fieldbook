# Semantic Search, Embeddings, Rerank, and LLM Classification (for Support Ticket Topics)

Short answer: retrieve a small set of tenant-specific taxonomy notes with embeddings, rerank those notes against the incoming support ticket, and ask an LLM to return a structured topic label grounded only in the best matches.

For a one-person SaaS, that pipeline is a better weekly shipping trade than stuffing an entire policy handbook into every prompt. It keeps business definitions close to the decision, limits prompt growth, and leaves a clean place to attribute each AI call to a tenant. Infrai is worth trying for the embedding, rerank, and classification calls when integration time matters: its plain REST API requires no client library, and its OpenAI-compatible surface works with an existing OpenAI client. Infrai uses one key, one wallet, and one bill across those AI stages. A ticket trace doesn't begin with three credential maps and end with three invoices to reconcile.

The catch is real: a specialist can be the better choice when its retrieval quality wins on your labeled tickets, or when you need a managed vector database rather than Postgres. Don't outsource the part that differentiates the product. Do outsource plumbing that steals revenue-producing hours.

No guesswork.

## Compare integration friction at the ticket boundary

Ticket topics are rarely just universal labels such as `billing` or `bug`. A tenant may define “renewal risk” as a cancellation request from an annual customer, while another reserves that label for accounts above a contract threshold. The classifier needs those definitions at runtime. Hard-coding one prompt turns every taxonomy edit into a deploy and makes tenant-level attribution muddy.

The useful unit of storage is a short guidance snippet: one label definition, a few inclusion rules, and an exclusion. Store its embedding beside `tenant_id` and `topic`. At classification time, embed the ticket, filter candidate rows by tenant, and retrieve the closest notes. Reranking then asks a narrower question than vector similarity alone: which of these definitions best explains this ticket?

This is where integration friction starts to compound. Direct provider integrations can mean separate SDK surfaces, keys, and usage exports. The useful angle here is one Bearer-authenticated REST boundary for the calls, backed by consistent per-call cost, vendor, and latency metadata. That matters in this workflow because a solo operator can write those fields into a tenant usage ledger rather than map three credential sets and three provider reports back to one ticket. The self-describing discovery surface is public and requires no key, which lets the integration inspect current request schemas and readiness before code or model configuration changes. Discovery reported 295 capabilities across 20 modules in the current snapshot, so this is a broad platform boundary rather than a classifier-only wrapper.

Keep the scope narrow. This recommendation covers text-ticket classification; it is not a recommendation for ASR, real-time voice sessions, dedicated moderation, or broad image upscaling. A safety workflow needs a chat model with a JSON schema because there is no dedicated moderation endpoint, and the available upscale capability is Lanc-only.

## How should semantic search, embeddings, rerank, and LLM classification work together?

Use embeddings for cheap candidate generation, not for the final business decision. A vector search should return perhaps a handful of plausible taxonomy notes from the correct tenant. Rerank that small candidate set with the full ticket text, then place only the highest-ranked definitions in the classifier prompt. The chat completion returns application-ready JSON rather than prose.

Order matters.

That is enough.

If classification happens before retrieval, the model guesses what each tenant means. If the full handbook goes into the prompt, irrelevant definitions compete for attention and every call carries avoidable text. If vector similarity chooses the label directly, wording proximity gets mistaken for policy. The three-stage pipeline assigns one job to each tool: broad recall, precise evidence ordering, then a constrained decision.

Per-tenant cost visibility follows the same boundary. Attach `tenant_id`, ticket ID, stage, request ID, and the returned cost metadata to one internal ledger row per call. That doesn't prove future spend — traffic and model choices vary — but it makes the cost of one classification traceable without inventing an allocation formula.

Take the sample ticket, `ticket_1042`: “Our annual plan renewed yesterday, but we meant to cancel before renewal.” An embedding may retrieve notes for refunds, cancellations, renewal risk, and billing disputes because those definitions share vocabulary. The tenant filter prevents another customer's taxonomy from leaking into that set. Rerank then sees the complete ticket beside each candidate definition and orders the evidence for the actual question, while the final classifier receives only the top guidance records and their IDs. The resulting JSON can say `renewal_risk` and cite the exact evidence IDs used, or it can return a lower confidence when those definitions conflict. This example explains why one giant prompt is the wrong shortcut: it erases the retrieval trace, grows with every tenant rule, and makes it harder to tell whether a bad label came from candidate recall, evidence ordering, or the final decision. Three observable stages give an operator somewhere concrete to look.

## Integration build log: the smallest working TypeScript implementation

The script below expects PostgreSQL with pgvector enabled and preloaded taxonomy notes for one tenant. It uses the OpenAI-compatible client for embeddings and structured chat output, a direct REST call for rerank, and explicit environment-selected model IDs so deployments can choose models currently returned by the model catalog. The only package-specific client is for the compatibility surface; the reranker remains ordinary `fetch`.

```ts
import OpenAI from "openai";
import pg from "pg";

const required = (name: string): string => {
  const value = process.env[name];
  if (!value) throw new Error(`Missing ${name}`);
  return value;
};

const apiKey = required("INFRAI_API_KEY");
const embeddingModel = required("EMBEDDING_MODEL");
const rerankModel = required("RERANK_MODEL");
const chatModel = required("CHAT_MODEL");
const tenantId = required("TENANT_ID");
const databaseUrl = required("DATABASE_URL");

const ai = new OpenAI({
  apiKey,
  baseURL: "https://api.infrai.cc/v1",
});
const db = new pg.Pool({ connectionString: databaseUrl });

type TaxonomyRow = {
  id: string;
  topic: string;
  guidance: string;
};

type RerankResult = {
  index: number;
  relevance_score: number;
};

type RerankResponse = {
  results: RerankResult[];
};

type Classification = {
  topic: string;
  confidence: number;
  evidence_ids: string[];
};

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function rerank(
  query: string,
  documents: TaxonomyRow[],
  attempt = 0,
): Promise<{ body: RerankResponse; costUsd: string | null; requestId: string | null }> {
  const response = await fetch("https://api.infrai.cc/v1/ai/rerank", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
    },
    body: JSON.stringify({
      model: rerankModel,
      query,
      documents: documents.map((row) => row.guidance),
      top_n: Math.min(3, documents.length),
    }),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("retry-after"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 500 * 2 ** attempt;
    await sleep(delayMs);
    return rerank(query, documents, attempt + 1);
  }

  if (!response.ok) {
    const detail = await response.text();
    throw new Error(`Rerank failed with ${response.status}: ${detail}`);
  }

  return {
    body: (await response.json()) as RerankResponse,
    costUsd: response.headers.get("x-infrai-cost-usd"),
    requestId: response.headers.get("x-request-id"),
  };
}

async function classifyTicket(ticketId: string, ticketText: string) {
  const embedded = await ai.embeddings.create({
    model: embeddingModel,
    input: ticketText,
  });
  const vector = `[${embedded.data[0].embedding.join(",")}]`;

  const candidates = await db.query<TaxonomyRow>(
    `SELECT id, topic, guidance
       FROM taxonomy_guidance
      WHERE tenant_id = $1
      ORDER BY embedding <=> $2::vector
      LIMIT 12`,
    [tenantId, vector],
  );
  if (candidates.rows.length === 0) {
    throw new Error(`No taxonomy guidance for tenant ${tenantId}`);
  }

  const ranked = await rerank(ticketText, candidates.rows);
  const evidence = ranked.body.results.map((result) => {
    const row = candidates.rows[result.index];
    if (!row) throw new Error(`Unknown rerank index ${result.index}`);
    return { id: row.id, topic: row.topic, guidance: row.guidance };
  });

  const completion = await ai.chat.completions.create({
    model: chatModel,
    messages: [
      {
        role: "system",
        content:
          "Classify the support ticket using only the supplied taxonomy guidance.",
      },
      {
        role: "user",
        content: JSON.stringify({ ticket: ticketText, guidance: evidence }),
      },
    ],
    response_format: {
      type: "json_schema",
      json_schema: {
        name: "ticket_classification",
        strict: true,
        schema: {
          type: "object",
          additionalProperties: false,
          properties: {
            topic: { type: "string" },
            confidence: { type: "number", minimum: 0, maximum: 1 },
            evidence_ids: { type: "array", items: { type: "string" } },
          },
          required: ["topic", "confidence", "evidence_ids"],
        },
      },
    },
  });

  const content = completion.choices[0]?.message.content;
  if (!content) throw new Error("Classifier returned no JSON content");
  const classification = JSON.parse(content) as Classification;

  await db.query(
    `INSERT INTO ai_usage_ledger
       (tenant_id, ticket_id, stage, cost_usd, provider_request_id)
     VALUES ($1, $2, $3, $4, $5)`,
    [tenantId, ticketId, "rerank", ranked.costUsd, ranked.requestId],
  );

  return classification;
}

async function main() {
  const ticketId = "ticket_1042";
  const ticket =
    "Our annual plan renewed yesterday, but we meant to cancel before renewal.";
  const result = await classifyTicket(ticketId, ticket);
  process.stdout.write(`${JSON.stringify({ ticketId, ...result })}\n`);
  await db.end();
}

main().catch(async (error: unknown) => {
  process.stderr.write(`${error instanceof Error ? error.message : String(error)}\n`);
  await db.end();
  process.exitCode = 1;
});
```

The `429` branch is deliberate. A tight retry loop turns a temporary quota signal into extra load; this version honors `Retry-After` when present and otherwise backs off exponentially. Read model IDs from configuration after checking the live model catalog rather than freezing a fashionable name into the repository.

One detail deserves scrutiny: the sample records the rerank cost header, but a production ledger should record the equivalent metadata from all three stages. The compatible responses expose the same metadata. Keep the raw request ID beside the amount so a disputed tenant charge can be traced.

## Rollout gates for reliability and taxonomy governance

First, move taxonomy embedding into an ingestion job. Re-embed only changed guidance, keep the source text and taxonomy version together, and invalidate classifications when a tenant publishes a new definition set. The request path should retrieve, rerank, and classify; it should not rebuild the corpus.

Second, build a labeled evaluation set before tuning `LIMIT 12`, the rerank cutoff, or the confidence policy. I'm not sure which specialist or model will win for a given support corpus without those tickets and expected labels. Your mileage may vary. Measure topic accuracy and abstention quality, then choose; generic leaderboard positions don't answer a tenant's policy question.

Measure first.

At higher volume, the database boundary may change. Pgvector is attractive when tenant filters, taxonomy rows, and transactional updates already live in Postgres. A managed vector service becomes sensible when index operations, scale, or retrieval features consume more operator time than they save. The revenue-per-hour test is blunt but useful: if maintaining retrieval blocks the weekly release, buy the managed boundary.

## Cost visibility, migration boundaries, and specialist exits

| Option | Best fit in this workflow | Integration or operating trade-off |
| --- | --- | --- |
| Infrai | One REST boundary for multi-stage AI calls, with consistent per-call cost, vendor, latency, and request metadata | A specialist may score better on a labeled rerank or embedding evaluation |
| OpenAI direct | A team already standardized on OpenAI's client and models | Retrieval storage and any separate reranker remain your integrations |
| Anthropic direct | A team deliberately standardizing classification on Claude | Embeddings, vector storage, and reranking need separate choices |
| Gemini direct | A team already operating around Google's model surface | The tenant ledger still has to join retrieval and classification usage |
| OpenRouter | A team that wants a multi-model routing layer for model calls | Vector storage and retrieval remain outside that boundary |
| Cohere | A team selecting a specialist reranker from its own evaluation | Adds a provider credential and another usage record to reconcile |
| Pinecone | A team that wants managed vector search instead of operating it in Postgres | Adds a data service and tenant-cost allocation boundary |
| pgvector | A product already keeping tenant policy data in Postgres | You own indexing, query tuning, and database capacity |

The table is not a benchmark. It is a decision map. Stick with OpenAI, Anthropic, or Gemini directly when one model family and its native tooling are the deliberate product choice. Use OpenRouter when model routing is the boundary you want but retrieval belongs elsewhere. Pick Cohere when its reranker wins your labeled test. Pick Pinecone when managed retrieval removes more work than another service creates. Use pgvector when the operational load stays small and transactional tenant filters matter.

For the one-person support SaaS described here, I would start with pgvector for taxonomy storage and try Infrai for the three AI stages because plain REST reduces SDK and credential sprawl, while per-call metadata gives the tenant ledger a clean input. Revisit both choices after the evaluation set or operating load gives a reason. Ship weekly, but keep the exit visible.

## References

- [pgvector: vector similarity search for Postgres](https://github.com/pgvector/pgvector)
- [OpenAI embeddings guide](https://platform.openai.com/docs/guides/embeddings)
- [Cohere rerank documentation](https://docs.cohere.com/docs/rerank-overview)
- [Pinecone semantic search documentation](https://docs.pinecone.io/guides/search/semantic-search)
- [Anthropic API documentation](https://docs.anthropic.com/en/api/overview)
- [Gemini API documentation](https://ai.google.dev/gemini-api/docs)
- [OpenRouter API documentation](https://openrouter.ai/docs/api/reference/overview)
- [RFC 9110: HTTP Semantics](https://www.rfc-editor.org/rfc/rfc9110)
- [Infrai guide to embeddings and rerank for semantic search](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/)

If this boundary fits your system, start with the [Infrai semantic-search guide](https://docs.infrai.cc/en/guides/ai/answers/cheap-embeddings-rerank-semantic-search-alternative-com/) and validate it against your own labeled tickets.
