# Logistics Chatbot API 2026: Multi-Model Gateway Streaming With JSON Schema Acceptance

| Choice | Quality control | Latency consequence |
| --- | --- | --- |
| One candidate, validate before saving | One extraction policy to check | No second generation on the accepted path |
| Sequential second candidate | Another attempt when validation rejects a draft | Rejected drafts wait for another generation |
| Parallel candidates | Multiple drafts available for comparison | Both calls consume resources; selection still needs a rule |

TL;DR: Start with one model candidate behind a small application interface. Validate its catalog proposal before saving it. Add a second candidate only when a defined quality failure justifies the extra wait. For a logistics chatbot enriching messy product descriptions, streaming can show progress, but the saved record needs a complete, checked result. A unified multi-model gateway can centralize transport; the application still owns acceptance.

This is a decision about when a proposed catalog field becomes usable. It also answers the practical runtime question: where streaming, JSON Schema, and tool calling belong when a chat response can change business data. The comparison below concerns execution policies, with no assumption that any particular model or gateway implements identical semantics.

## How should a multi-model chatbot API compare streaming and JSON Schema results?

Consider this invented input for a test fixture: `2 cartons; 12 bottles/carton; 500 mL each; gross wt TBD`. The proposed record must keep package count, contents per package, and per-item volume separate. Treating 500 mL as the shipment volume is a semantic error even if the response is valid JSON. Guessing a gross weight creates another failure. Leave it unknown.

Those numbers are example data, not benchmark results. For this fixture, an acceptance rule could require two cartons, twelve bottles per carton, a per-bottle volume of 500 mL, and no asserted gross weight. Make the field definitions explicit before comparing candidates; otherwise a model that guesses more fields can appear more useful than one that preserves uncertainty. Include the source text with the review view so a person can check the proposal without reconstructing the conversation.

Preserve the distinction.

The first decision criterion is accepted field quality: does the output express the meaning supported by the description? JSON Schema supplies a vocabulary for structural constraints. It cannot, by itself, establish that a generated value is grounded in a supplier's text. Use application checks for known relationships and route ambiguous descriptions to review. A schema-valid record is an intermediate result. The second criterion is elapsed time until the record can be accepted. Measure that separately from time to the first displayed text. A fast opening sentence has little operational value if the warehouse user must wait for the final dimensions before continuing. Conversely, a conversational explanation may be useful before a background enrichment proposal is ready, provided the interface clearly identifies its pending state. In the bottle fixture, the chatbot could immediately acknowledge receipt while the proposal remains pending; an acceptance check would then preserve the distinction between the two cartons and the bottles inside them. The acknowledgment doesn't authorize a write.

Wait for acceptance.

For a one-person SaaS, the revenue-per-hour lens favors a policy that is easy to inspect and change. Ship weekly. Outsource the undifferentiated transport work where that reduces maintenance, and keep the catalog acceptance rules close to the product. Do not add a parallel comparison service just to display an impressive first token. Its selection logic and review burden need to earn their place.

## Keep streaming separate from catalog acceptance

Use two application states: a visible preview and an accepted proposal. Streaming feeds the preview. Only the completed response enters parsing, validation, and the eventual write path. An interrupted response leaves the catalog untouched; the UI can retain a visibly incomplete preview or clear it according to its interaction design.

A unified API is useful as a transport boundary, but a shared request shape is insufficient evidence of interchangeable behavior. For each configured candidate, verify the exact model and route used for streaming, structured output, and tools. Check the selected gateway's documentation and exercise the combinations the application needs. Do not infer support from a different candidate using the same endpoint. This is a verification requirement, not a claim about a particular service.

Tool calling needs its own acceptance boundary. Treat a generated tool request as proposed input. Parse it, validate the arguments, authorize the action for the current user, and only then invoke application code. A catalog description is untrusted material; it must not acquire permission to update records by containing instructions. OWASP's LLM application guidance identifies prompt injection and excessive agency as relevant risks. For enrichment, a narrow proposal-producing operation is easier to reason about than unrestricted write access.

Transport success is not acceptance.

An HTTP success response can still carry a proposal that your business rules reject. Keep those outcomes separate in application telemetry so a network dashboard cannot silently become your quality dashboard.

## A typed boundary for one candidate and a second attempt

The following TypeScript example is application-owned orchestration for a server runtime. Its interfaces are illustrative, not a vendor SDK or a claim that a public route accepts these fields. The adapter must assemble a completed response and honor cancellation; the decoder must parse JSON, validate the chosen schema, and apply catalog-specific checks before returning a proposal.

```ts
type Proposal = {
  packageCount: number | null;
  unitsPerPackage: number | null;
  itemVolumeMl: number | null;
  grossWeightKg: number | null;
};

type Candidate = {
  id: string;
  complete(input: string, signal: AbortSignal): Promise<string>;
};

type Verdict =
  | { ok: true; proposal: Proposal }
  | { ok: false; reason: string };

type Outcome =
  | { status: "proposed"; candidate: string; proposal: Proposal }
  | { status: "review"; reasons: string[] };

async function proposeEnrichment(
  input: string,
  candidates: readonly Candidate[],
  signal: AbortSignal,
  decodeAndCheck: (raw: string, source: string) => Verdict,
): Promise<Outcome> {
  const reasons: string[] = [];

  for (const candidate of candidates.slice(0, 2)) {
    signal.throwIfAborted();
    const raw = await candidate.complete(input, signal);
    signal.throwIfAborted();
    const verdict = decodeAndCheck(raw, input);

    if (verdict.ok) {
      return {
        status: "proposed",
        candidate: candidate.id,
        proposal: verdict.proposal,
      };
    }
    reasons.push(`${candidate.id}: ${verdict.reason}`);
  }

  return { status: "review", reasons };
}
```

Pass one candidate initially. Passing two enables a sequential second attempt after a validation rejection. Transport failures propagate to the caller instead of being silently reclassified as poor extraction. The caller owns its deadline, user-facing error handling, and any separately bounded retry policy. No database write occurs in this function.

There is a deliberate limitation: the first proposal that passes the supplied checks wins. If those checks miss a plausible but unsupported weight, the second candidate never runs. Multiple models cannot repair a weak acceptance rule merely by being available. Improve the fixture and validator first, then decide whether another attempt changes the outcome. Also record when a second candidate returns a different value for the same evidence; disagreement is a review signal unless a documented domain rule resolves it.

## When is the runner-up policy better?

Sequential escalation is the runner-up when one candidate repeatedly produces reviewable ambiguity and another has demonstrated better accepted results on those same descriptions. Establish that with a labeled test set before enabling escalation. The trade is explicit: rejected first attempts take longer, and every added configuration needs maintenance. No performance advantage is assumed here.

Parallel candidates fit a narrower case. Use them when the product has a strict completion deadline and your measurements show that waiting to start a second attempt misses it. Define the selection rule beforehand. Taking the first finished response privileges speed; waiting for agreement privileges a form of consistency that can still be wrong. Neither policy establishes factual correctness without checking the source.

Human review is a valid outcome.

For an ambiguous unit or unsupported gross weight, preserving a missing value can protect quality better than spending another generation trying to make the record look complete.

## Ship a measurable acceptance policy

Keep fixtures for package-versus-item confusion, absent units, contradictory dimensions, and instruction-like text inside descriptions. These are proposed test categories for this workflow. For each fixture, specify the acceptable values, required unknowns, and circumstances that require review. Test streaming interruption and cancellation separately from completed-output quality, including the assertion that neither can commit a partial proposal.

Before a weekly release, replay those fixtures against the configured candidates and compare accepted field quality with elapsed time to an accepted proposal. Track review frequency and second-attempt frequency alongside latency. Log the candidate identifier, acceptance-rule version, and rejection category; avoid copying full supplier descriptions into routine logs when a record identifier will support investigation. A changed schema, prompt, or candidate configuration deserves another replay because it changes the behavior under review.

The deployment switch should choose an execution policy, with a straightforward return to the previous configuration if measured acceptance deteriorates. Keep the initial write path behind review until the domain checks justify automatic acceptance. For runtime validation machinery or stream handling, prefer maintained components with documented behavior over custom infrastructure whose upkeep competes with product delivery.

The useful cost question is how much engineering and operator attention each accepted record requires. Include model calls, review time, and ongoing maintenance in that judgment. Published unit prices alone cannot answer it, and they should not drive a claim that a given route is cheapest for an unmeasured workload.

Start with one candidate and a visible review state. Add escalation when evidence shows that its quality gain is worth the added latency. That gives the logistics chatbot a clear contract: conversation can arrive incrementally; catalog changes arrive only after acceptance.

## References

The technical sources below support the schema vocabulary, cancellation interface, and security boundaries discussed here. The sample fixture and policy choices are illustrative engineering proposals, not measured results.

## Further reading

- JSON Schema validation specification: https://json-schema.org/draft/2020-12/json-schema-validation
- MDN, AbortSignal and cancellation: https://developer.mozilla.org/en-US/docs/Web/API/AbortSignal
- OWASP Top 10 for LLM Applications: https://owasp.org/www-project-top-10-for-large-language-model-applications/
