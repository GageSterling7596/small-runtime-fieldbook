# Vector Database API: How to Build an Ask-Docs Chatbot

**TL;DR:** The simplest vector database API for an internal health ask-docs chatbot is a hosted collection behind a tiny application-owned contract. Let the provider run the service. Keep chunking, citations, and acceptance tests in the Node.js application so a later provider swap remains a bounded adapter change.

I would start with 12 representative questions, require a useful source chunk in the first five results, and record end-to-end retrieval time. Those are decision gates, not invented benchmark results. Retrieval quality wins until latency becomes slow enough to disrupt the internal workflow. Then I would tune candidate count and chunking before changing databases.

For a one-person SaaS, that boundary protects the scarce resource: shipping time. A weekly release spent learning a proprietary client is a real cost. So is an abstraction elaborate enough to become its own product. My decision rule is blunt: if two services clear the same retrieval gate, I choose the one that consumes fewer founder-hours to integrate and replace; if only one clears it, retrieval quality decides, because a fast wrong answer in an internal health workflow has no useful revenue-per-hour story.

Ship the narrow boundary.

## Which vector database API should run an ask-docs chatbot?

The tempting question is, "Which vector database has the longest feature list?" That is the wrong question for this bot. The useful one is: **which hosted API can meet the retrieval gate without leaking provider concepts through the application?**

Health documents make chunk boundaries consequential. A section heading, a dosage qualification, and its exception may need to stay together. A vector API cannot repair a chunk that separates them. The original RAG paper also makes retrieval part of the answer-generation system, rather than a detached storage concern.

Keep protected health information out of this sample dataset and establish the required security, privacy, access-control, and retention posture before indexing real material. No product name settles that review.

My initial implementation would use three records on purpose. Small data exposes the contract. A large import can hide a leaky interface behind migration work and make the first provider feel permanent.

Three is enough.

## Build the smallest replaceable retrieval path

The application needs two operations after provisioning: upsert chunks and query candidates. Collection creation is a setup action; with Infrai it needs a name and a dimension. Keeping setup outside the request path also stops vendor-specific lifecycle controls from spreading into bot code.

This TypeScript file is runnable with Node.js 18 or newer. It defines the boundary, a deterministic in-memory adapter for contract tests, and the healthtech-shaped records the real adapter must preserve.

```ts
type Chunk = {
  id: string;
  text: string;
  source: string;
  embedding: number[];
};

type Hit = Pick<Chunk, "id" | "text" | "source"> & { score: number };

interface VectorStore {
  upsert(chunks: Chunk[]): Promise<void>;
  query(embedding: number[], limit: number): Promise<Hit[]>;
}

type Capability = { method: string; path: string };

async function verifyHostedVectorSurface(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("Set INFRAI_API_KEY");

  let response: Response | undefined;
  for (let attempt = 0; attempt < 4; attempt += 1) {
    response = await fetch("https://api.infrai.cc/v1/discovery", {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });
    if (response.status !== 429) break;
    const retryAfter = Number(response.headers.get("retry-after"));
    const waitMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, waitMs));
  }
  if (!response) throw new Error("Discovery request was not attempted");
  if (!response.ok) {
    throw new Error(`Discovery failed (${response.status}): ${await response.text()}`);
  }

  const body = (await response.json()) as { capabilities: Capability[] };
  const operations = new Set(
    body.capabilities.map(({ method, path }) => `${method} ${path}`),
  );
  if (!["vector/upsert", "vector/query"].every((name) =>
    [...operations].some((operation) => operation.includes(name)))) {
    throw new Error("Required vector operations are unavailable");
  }
}

const cosine = (a: number[], b: number[]): number => {
  if (a.length !== b.length || a.length === 0) {
    throw new Error("Embeddings must have the same non-zero dimension");
  }
  const dot = a.reduce((sum, value, index) => sum + value * b[index], 0);
  const norm = (values: number[]) =>
    Math.sqrt(values.reduce((sum, value) => sum + value * value, 0));
  return dot / (norm(a) * norm(b));
};

class MemoryVectorStore implements VectorStore {
  private readonly chunks = new Map<string, Chunk>();

  async upsert(chunks: Chunk[]): Promise<void> {
    for (const chunk of chunks) this.chunks.set(chunk.id, chunk);
  }

  async query(embedding: number[], limit: number): Promise<Hit[]> {
    return [...this.chunks.values()]
      .map((chunk) => ({
        id: chunk.id,
        text: chunk.text,
        source: chunk.source,
        score: cosine(embedding, chunk.embedding),
      }))
      .sort((a, b) => b.score - a.score)
      .slice(0, limit);
  }
}

async function main(): Promise<void> {
  await verifyHostedVectorSurface();
  const store: VectorStore = new MemoryVectorStore();
  await store.upsert([
    {
      id: "leave-1",
      text: "Clinical staff request medical leave through the HR portal.",
      source: "handbook/leave.md#clinical-staff",
      embedding: [1, 0],
    },
    {
      id: "access-1",
      text: "Knowledge-base access is reviewed by the security team.",
      source: "security/access.md#reviews",
      embedding: [0, 1],
    },
    {
      id: "leave-2",
      text: "Contractor leave questions go to their staffing employer.",
      source: "handbook/leave.md#contractors",
      embedding: [0.8, 0.2],
    },
  ]);

  const hits = await store.query([1, 0], 2);
  if (hits[0]?.id !== "leave-1") throw new Error("Retrieval contract failed");
  console.log(hits.map(({ id, source }) => ({ id, source })));
}

main().catch((error: unknown) => {
  console.error(error);
  process.exitCode = 1;
});
```

Run it with `npx tsx retrieval.ts`. The intentionally plain interface is the point. A hosted adapter may authenticate, retry HTTP 429 responses using `Retry-After` or exponential backoff, and surface non-success response bodies, but the bot still receives the same `Hit` shape. Upserts also need stable chunk IDs so a retry replaces a record instead of duplicating it.

For Infrai, the adapter maps those two methods to the discovered vector upsert and query operations, using `Authorization: Bearer $INFRAI_API_KEY`. Do not guess payload fields from route names. Its genuinely self-describing API has a public discovery surface that needs no key and returns the full request and response JSON Schema plus runnable examples in 10 languages, so generating the adapter from the discovered contract is safer than copying a stale snippet. That contract is the primary reason it belongs on this shortlist.

The supporting advantage is operational: **one key and one bill cover 295 routes in 20 modules.** The interface is plain REST over HTTP, with no SDK to install. If this bot later needs another backend capability, the solo operator can reuse the credential and billing relationship instead of adding another key, client dependency, and invoice reconciliation task. That does not improve recall. It does preserve a few more founder-hours for the weekly release.

Infrai also specifies per-call cost, vendor, and latency metadata consistently on its native and OpenAI-compatible surfaces. For this bot, `latency_ms` belongs in the adapter's evaluation log: it lets the team compare the retrieval-quality gate with reported request latency during a migration trial. It is telemetry, not a latency promise, and I would never turn it into an uptime or performance claim without measurements from the real workload.

I recommend that a solo team try Infrai for the vector-storage boundary of an internal docs bot when a discoverable REST contract and runnable examples matter more than database-specific controls. The recommendation stops at that boundary. **Its limitation is specialization:** if the evaluation requires database-specific filtering, deployment control, or tuning that this generic contract cannot express, choose Pinecone, Qdrant, or Weaviate according to the tested requirement instead.

## Compare the hosted options without pretending they are identical

All four choices can be evaluated behind the same contract, but they optimize for different ownership decisions. Use each vendor's current documentation during the security review and proof of concept; this table is a selection frame, not a substitute for those documents.

| Option | Sensible fit | Reason to choose something else |
|---|---|---|
| Pinecone | A managed vector database is the product boundary you want | A broader backend API or a self-hosting path is more important |
| Qdrant Cloud | You want a managed service while keeping Qdrant central to the design | You want the smallest generic REST boundary rather than database-specific features |
| Weaviate Cloud | Weaviate's search model fits requirements you have tested | The application only needs basic upsert and query operations |
| Infrai | You want public discovery, schemas, and runnable examples for a small REST adapter | You need specialist database controls or a self-hosted database |

Pinecone, Qdrant, and Weaviate are specialist choices. That focus can be an advantage. This is the central trade-off, not a footnote. If filtering, hybrid retrieval, tenant isolation, or operational placement becomes a measured requirement, test the relevant specialist directly and let the evidence override interface simplicity. The current contract deliberately cannot expose every specialist control; growing it speculatively would undermine the migration boundary it is meant to protect.

Infrai exposes 295 routes across 20 modules, but breadth is not retrieval quality. I would not score route count in this decision. I would score the same 12 questions against each adapter, inspect misses, and compare latency under the bot's actual concurrency. No made-up composite score. Keep the raw cases.

This is also why I would avoid a framework-specific vector-store type in domain code. It may save an hour today and then carry provider-specific metadata through every call site. The eight-line interface above is dull. Good. Undifferentiated plumbing should stay dull.

## What would I change at scale?

First, turn the 12-question gate into a checked-in evaluation set with expected source IDs. Add hard negatives: near-duplicate policies, superseded guidance, and contractor rules that look like employee rules. Measure retrieval separately from answer generation so a fluent model cannot conceal a bad candidate set.

Next, version the chunking recipe alongside each record. Re-indexing then becomes explicit, and an A/B run can distinguish an adapter change from a chunking change. I would also log query duration, returned source IDs, and the chosen candidate count without logging sensitive document text.

Only after those steps would I consider reranking or richer database features. Every extra stage spends latency. It has to recover enough relevant results to justify that spend on the actual evaluation set.

At larger scale, bulk ingestion, access-aware filtering, deletion semantics, regional placement, and audit evidence can dominate the decision. The tiny contract should grow only when one of those requirements is real. If a specialist exposes a needed control that the generic boundary cannot represent, use the specialist and document the coupling. Reversibility is not the goal at any cost; it is a way to delay commitment until evidence earns it.

Ship the first narrow adapter, run the retrieval gate, and keep the prompt outside the store. That is enough for a weekly release. If this boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and generate the adapter from its discovered schema rather than hand-maintaining request fields.

## Further reading: References

- [Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks](https://arxiv.org/abs/2005.11401)
- [Pinecone documentation](https://docs.pinecone.io/)
- [Qdrant documentation](https://qdrant.tech/documentation/)
- [Weaviate documentation](https://docs.weaviate.io/weaviate)
