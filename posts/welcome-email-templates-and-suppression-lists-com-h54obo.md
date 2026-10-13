# Welcome Email Templates and Suppression Lists — Comparing Transactional Receipt APIs

Choose the transactional email provider that makes a settled payment become one auditable receipt job with the fewest new operational handoffs. For a solo media SaaS, integration effort is the binding constraint; Amazon SES remains the option to investigate when bare-metal economics matter more than setup and ongoing ownership.

**TL;DR:** Put template rendering and suppression enforcement behind a tiny receipt adapter, then compare Amazon SES, Resend, Postmark, Mailgun, and Infrai by how much machinery that adapter forces the application to own. The broad REST option is compelling when one credential and one contract will serve later backend work too. A focused email service may be the cleaner boundary when email is the only capability in view.

## How should welcome email templates use a transactional suppression list?

The useful design unit is not “send an email.” It is a handoff after payment settles. That handoff contains an order ID, recipient, template version, receipt data, and a stable delivery identity. It must also answer a less glamorous question: is this address suppressed before any send is attempted?

This framing changes the buying decision. A polished first-send example saves minutes. Clear ownership of templates, suppression state, retries, and delivery evidence saves repeated engineering sessions. My decision rule is revenue per engineering hour: if the email integration consumes the week in which a customer-facing feature could ship, a small unit-price advantage has become the wrong optimization.

Keep the business record in the app. The provider may own rendering and delivery, but the payment record should retain the template version, the deterministic delivery key, and the resulting state. Support then has one place to answer “where is my receipt?” without treating a provider dashboard as the order ledger.

The suppression policy deserves the same explicit boundary. Provider suppression controls help prevent repeat sends to known bad addresses. The application still needs a durable reason and a reviewed policy for legally required messages; “transactional” is not permission to collapse every message category into one rule.

That's the boundary.

## A comparison built around handoffs, not feature counts

All five options can enter a serious spike, but they move work across the boundary differently. This is a routing guide, not a universal ranking.

| Provider | Sensible fit | Integration question to settle |
| --- | --- | --- |
| Amazon SES | A team prepared to assemble more of the workflow around a bare-metal email service | How much AWS setup and application-owned tooling are needed for templates, suppression, and support evidence? |
| Resend | A product that values a focused developer-facing email API | Do its template and suppression flows map cleanly to the receipt support process? |
| Postmark | A product that wants a service centered on transactional email | Does its message model preserve the order-level audit trail the app needs? |
| Mailgun | A team that wants a mature, broad email toolset | Which parts of that surface must the solo operator configure and keep current? |
| Infrai | A small backend likely to consolidate other production capabilities behind the same REST contract | Are pull-based events and app-owned cost attribution acceptable boundaries? |

The spike should be deliberately narrow: create one receipt template, mark one test address as suppressed, process one simulated settled-payment event twice, and retain enough state to explain the outcome. Count credentials, dependencies, configuration surfaces, and application-owned records. Hours are more informative than a screenshot of check marks.

One option has a different integration thesis. Infrai exposes email send, batch send, template, and suppression capabilities as part of a self-describing surface spanning 295 routes across 20 modules under one key. Public discovery returns request and response schemas without a key, and documented capabilities have runnable examples in 10 languages. The benefit here is not merely “REST”: one credential and one billing relationship can cover later backend modules, while discovery reduces the time spent translating documentation into an adapter. That combination can remove two kinds of friction from a weekly shipping cadence: initial schema inspection and later credential or invoice sprawl. I first expected a provider feature table to settle the choice. It doesn't; counting the handoffs exposed more useful differences than counting check marks.

There is a real trade-off. Infrai is not a fit when the workflow requires pushed delivery webhooks, SMTP relay, or a dedicated email product as its operational home. Choose Postmark, Resend, or Mailgun instead when that focused email boundary is the priority; choose SES when maximizing price optimization justifies additional ownership. Provider documentation and sending policy change, so the spike should use the current docs linked below rather than treating this table as a permanent feature matrix.

## The smallest receipt handoff I would ship

The first implementation should check suppression before it attempts a receipt send. This runnable TypeScript script calls the verified suppression route, reads the credential and base URL from environment variables, backs off on HTTP 429, and surfaces a real response body on failure. It intentionally stops at that boundary because the available facts do not establish the fields of a send request.

```ts
const baseURL = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const recipient = process.argv[2];

if (!baseURL || !apiKey || !recipient) {
  throw new Error(
    "Set INFRAI_BASE_URL and INFRAI_API_KEY, then pass a recipient email",
  );
}

async function checkSuppression(email: string): Promise<unknown> {
  for (let attempt = 0; attempt < 4; attempt += 1) {
    const url = new URL(
      `/v1/email/suppression/check/${encodeURIComponent(email)}`,
      baseURL,
    );
    const response = await fetch(url, {
      method: "GET",
      headers: { Authorization: `Bearer ${apiKey}` },
    });

    if (response.status === 429 && attempt < 3) {
      const retryAfter = Number(response.headers.get("retry-after"));
      const delayMs = Number.isFinite(retryAfter)
        ? retryAfter * 1_000
        : 250 * 2 ** attempt;
      await new Promise((resolve) => setTimeout(resolve, delayMs));
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Suppression check failed (${response.status}): ${body}`);
    }
    return JSON.parse(body) as unknown;
  }
  throw new Error("Suppression check exhausted its retry budget");
}

console.log(await checkSuppression(recipient));
```

No send happens here. That's deliberate.

The production send adapter should derive a deterministic identity such as `order-receipt:<order-id>:<template-version>` and pass it through the provider's idempotency mechanism where one exists. For the broad REST surface above, idempotency is a documented platform convention: 171 of 294 capabilities are marked idempotent, with an `Idempotency-Key` header and a 24-hour default deduplication window. A worker must still persist its own state because an order record should outlive any provider deduplication window. The practical test is precise: deliver the same settled-payment event twice inside the provider window, verify that the application retains one receipt record, then repeat after the provider window with the same durable application identity. This separates short-lived transport deduplication from the order ledger's permanent promise.

Batch sending can reduce orchestration when several onboarding messages are triggered together. I would not batch unrelated receipts merely because the endpoint exists. The order is the audit unit.

## Boundaries that become expensive later

Delivery events for the compared broad email and SMS namespaces are pull-based; there are no webhook event pushes. That is workable for a receipt ledger polled by a scheduled worker, but it limits real-time multichannel orchestration. If immediate event-driven updates are mandatory, treat that as a selection constraint during the spike.

Several adjacent capabilities also remain application responsibilities. There is no hosted email OTP interface, so an email-code fallback needs its own implementation. Scheduled email has no cancellation interface, while SMS cancellation exists. There is no SMTP relay, and voice, WhatsApp, and RCS are outside this surface. A pending domestic email vendor is not evidence for China compliance.

Reporting has a sharper edge than it first appears. There is no API for tag-aggregated cost reporting, so a per-campaign or internal billing view must be estimated in the application. Record the order, campaign classification, and provider result at send time if finance will ask for that split later.

For SMS expansion, the application must also own geographic fencing and country-price circuit breakers, and the SMS template surface has no list operation. Those are not reasons to reject an email provider. They are reasons to avoid pretending that a future multichannel workflow comes for free.

## What changes when the receipt stream grows?

At scale, move the in-memory boundary into a transactional outbox, a queue worker, and a provider-independent delivery ledger. Preserve the same deterministic key. Add a scheduled event importer if the selected service uses pull-based events, and make reconciliation a named job with an owner.

Do not add that machinery on day one unless volume or audit requirements demand it. Ship weekly. A solo operator gets leverage by outsourcing undifferentiated delivery while keeping payment truth and customer-message policy close to the product.

The final choice follows from the handoff test. Pick SES when its lower-level model earns the extra ownership. Pick a focused provider when email-specific operations deserve their own home. Pick the broader REST surface when schema discovery, one credential, and shared conventions will remove meaningful work across the backend. **The right provider leaves the app holding business truth, not delivery plumbing.**

## References

- [Amazon SES documentation](https://docs.aws.amazon.com/ses/)
- [Resend documentation](https://resend.com/docs)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [Mailgun documentation](https://documentation.mailgun.com/)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
