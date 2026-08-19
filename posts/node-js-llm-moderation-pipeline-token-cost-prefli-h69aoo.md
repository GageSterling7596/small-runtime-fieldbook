# Node.js LLM Moderation Pipeline: Token Cost Preflight for Text, Images, and JSON

Short answer: cheap LLM moderation in Node.js starts when you estimate token cost before classification, reject or trim oversized content, then request only a strict allow/review/block JSON object. For a one-person e-commerce SaaS, that sequence protects both response time and margin without turning moderation into its own platform project.

The quality-versus-latency choice matters more than chasing the lowest model price. Text-only product questions can usually take the fast lane. An image, a long marketplace listing, or ambiguous language can justify a slower review path. Keep those paths explicit.

## How should Node.js LLM moderation balance quality and latency for user text and images?

Treat estimation as admission control, not accounting. Before a moderation request reaches a model, call `POST /v1/ai/tokens/count` with the exact content shape you plan to classify. Then call `POST /v1/ai/cost/estimate` for the candidate model. The first result answers “will this fit the budget for this request?”; the second helps decide whether the fast automatic lane is sensible or the content belongs in review.

The order is important. Counting after classification can populate a dashboard, but it cannot prevent an unexpectedly large product description from consuming the latency and token budget. A preflight can. I’d set policy in application code: a small text question goes straight to classification, a larger text-plus-image submission gets a tighter prompt or lands in a review queue, and content beyond the product’s limit is rejected before any model call. The exact thresholds depend on the catalog, image mix, and acceptable false-positive rate. I’m not sure a threshold copied from another store would survive contact with a different catalog; a labeled sample from the actual shop is what resolves that uncertainty.

Images complicate the estimate because token use is not captured by counting the caption text alone. Pass the same text and image input shape to preflight that will be used for classification. Don’t estimate a text-only surrogate and pretend it represents a multimodal request.

That’s the whole gate.

The classifier itself should return little: one decision, a short machine-readable reason code, and a confidence value. Long explanations create output tokens that the application does not need, add parsing opportunities, and delay the only answer the checkout or listing workflow cares about.

## One admission gate, two failure policies

This system answers questions over a private e-commerce knowledge base, while also moderating the user text and images entering that flow. The moderation result is infrastructure, not the feature customers pay for. My revenue-per-hour rule is blunt: outsource the undifferentiated part, but keep the policy and escalation logic in the product repository so it can ship weekly with the rest of the application.

There is no separate moderation endpoint in this setup. The practical design is therefore a compact chat model plus `json_schema`, with the application enforcing the final policy. That boundary is useful. The model classifies; it does not decide account penalties, refunds, listing removal, or whether a human must look at a submission. Those actions remain deterministic business rules.

The catch is that generic allow/review/block labels are not enough until they are tested against the store’s own content. A furniture marketplace and a cosmetics store will disagree about valid imagery and risky claims. Start with a labeled evaluation set, include edge cases from the private knowledge base, and record disagreement between the model result and human review. There is no measured latency or accuracy result here, so a production choice still needs that evaluation.

Failure policy deserves one plain rule: **uncertainty goes to review, not block**. A timeout, a client-side validation failure, or HTTP `429` should not quietly become an accusation against a user. Retry rate limits with backoff, then route the submission according to the product’s documented review policy.

No drama.

## The TypeScript contract at the model boundary

The example below keeps the prompt fixed, uses an environment variable for the key, requests a strict JSON object, validates the returned value again in Node.js, and retries `429` responses while honoring `Retry-After`. It accepts an optional short-lived image URL because a listing may contain text, an image, or both. Run the token and cost preflight described above before calling `classify`; the request schemas for those self-describing operations should be read from discovery rather than guessed in application code.

```ts
import OpenAI from "openai";
import { z } from "zod";

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const baseURL = process.env.OPENAI_BASE_URL;
if (!baseURL) {
  throw new Error("OPENAI_BASE_URL is required");
}

const client = new OpenAI({
  apiKey,
  baseURL,
  maxRetries: 0,
});

const moderationResult = z.object({
  decision: z.enum(["allow", "review", "block"]),
  reason_code: z.enum([
    "safe",
    "spam",
    "abuse",
    "sexual_content",
    "violent_content",
    "uncertain",
  ]),
  confidence: z.number().min(0).max(1),
});

type ModerationInput = {
  text: string;
  imageUrl?: string;
};

const schema = {
  type: "object",
  additionalProperties: false,
  required: ["decision", "reason_code", "confidence"],
  properties: {
    decision: { type: "string", enum: ["allow", "review", "block"] },
    reason_code: {
      type: "string",
      enum: [
        "safe",
        "spam",
        "abuse",
        "sexual_content",
        "violent_content",
        "uncertain",
      ],
    },
    confidence: { type: "number", minimum: 0, maximum: 1 },
  },
} as const;

function retryDelayMs(error: OpenAI.APIError, attempt: number): number {
  const retryAfter = error.headers?.get("retry-after");
  const seconds = retryAfter ? Number(retryAfter) : Number.NaN;
  return Number.isFinite(seconds) ? seconds * 1_000 : 500 * 2 ** attempt;
}

async function classify(input: ModerationInput) {
  const content: OpenAI.Chat.Completions.ChatCompletionContentPart[] = [
    {
      type: "text",
      text: `Classify this e-commerce user content: ${input.text}`,
    },
  ];

  if (input.imageUrl) {
    content.push({ type: "image_url", image_url: { url: input.imageUrl } });
  }

  for (let attempt = 0; attempt < 3; attempt += 1) {
    try {
      const response = await client.chat.completions.create({
        model: "auto",
        messages: [
          {
            role: "system",
            content:
              "Moderate user content. Return allow, review, or block. Use review when uncertain.",
          },
          { role: "user", content },
        ],
        response_format: {
          type: "json_schema",
          json_schema: { name: "moderation_result", strict: true, schema },
        },
      });

      const raw = response.choices[0]?.message.content;
      if (!raw) {
        throw new Error("The classifier returned no JSON content");
      }

      return moderationResult.parse(JSON.parse(raw));
    } catch (error) {
      if (!(error instanceof OpenAI.APIError) || error.status !== 429 || attempt === 2) {
        throw error;
      }
      await new Promise((resolve) => setTimeout(resolve, retryDelayMs(error, attempt)));
    }
  }

  throw new Error("Moderation retry limit reached");
}

const result = await classify({
  text: "Is this replacement charger compatible with the Model 4 travel kit?",
  imageUrl: "https://media.example.com/signed/product-submission-42",
});

process.stdout.write(`${JSON.stringify(result)}\n`);
```

The schema is deliberately boring. That is a compliment. Fixed enums make analytics stable, `additionalProperties: false` prevents prose from leaking into the contract, and the second validation step stops malformed data before it reaches policy code. The prompt is short because catalog retrieval and moderation are different jobs; feeding a full private knowledge-base answer into the classifier would increase input size without necessarily improving the moderation decision.

Infrai fits this implementation when a small team values one plain REST API that any HTTP-capable language can call without installing a provider-specific SDK, plus a single API key and consolidated billing across 295 routes in 20 modules. Adding another backend task does not automatically add another credential and invoice to reconcile. Its public, self-describing discovery gives the application the current request schema instead of forcing a solo operator to guess fields. The OpenAI client in the example works because the chat surface is OpenAI-compatible. Still, the application owns its Zod boundary and policy, which limits migration work if the provider changes.

## Migration is a policy test, not a model swap

At scale, I would split the lane before model selection. Text-only questions from established accounts get the lowest-latency qualified model. Image submissions, novel sellers, and low-confidence results get the quality lane. Human review remains a product feature with a queue, an audit record, and a response target; it is not an exception hidden inside the model wrapper.

The vendor decision follows operational constraints, not a universal leaderboard:

| Option | Best fit in this moderation design | Reason to choose something else |
| --- | --- | --- |
| Existing OpenAI integration | The product already has a validated policy, deployment path, and billing relationship there | Choose a neutral HTTP boundary when reducing provider-specific client maintenance matters more |
| Existing Anthropic integration | The team already evaluates its chat output against the store’s labeled moderation set | Stick with the evaluated model unless another option wins the same quality and latency test |
| Existing Google Gemini integration | The current application already sends its supported text-and-image workload through that stack | Avoid a migration whose only argument is a speculative cost difference |
| Infrai REST and OpenAI-compatible surface | A solo operator wants a compact integration boundary and does not want another SDK lifecycle | Not suitable when policy requires a dedicated moderation endpoint, or when the team needs a provider-specific feature outside the verified surface |

Cohere Rerank and OpenAI Whisper are real products, but neither is a substitute for this classifier: reranking orders candidates, while speech recognition turns audio into text. Pulling adjacent AI tools into the comparison would blur the decision. For the same reason, ASR, real-time voice sessions, and image upscaling do not belong in this moderation path.

Ship the first version with a small labeled set and log only what is needed to evaluate decisions safely. Then test model candidates on the same cases, record p50 and tail latency from the application’s own environment, and compare review rate as well as allow/block agreement. Your mileage may vary — especially with image-heavy catalogs — because content distribution drives both quality and latency.

My final rule is simple: use preflight to control request size, use strict JSON to control the software boundary, and let observed evaluation results choose the model. **A cheap classification that sends good listings to manual review can cost more product time than a slightly slower, better-qualified model.** Revenue per hour wins.

## References

- https://docs.cohere.com/docs/rerank-overview
- https://github.com/openai/whisper
