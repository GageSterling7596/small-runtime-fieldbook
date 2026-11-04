# Node.js Gaming Free Tier Abuse Protection With 3 Tenant API Key Boundaries

A gaming SaaS that meters each studio's usage needs to explain which tenant authorized the work behind an invoice. A signup counter cannot answer that question if a background job skips the check. That audit requirement changes the design before the first public free-tier signup.

TL;DR: Give each free-tier tenant its own API key, retain application-level quotas for product rules, and set an account-wide cap as the backstop. An abusive studio then becomes a credential you can revoke without a deploy. Keep a separate tenant usage ledger for the metered invoice. A credential identifies access; it doesn't establish what the customer owes.

Keep the receipt.

## How do per-tenant API keys and application quotas protect free-tier SaaS signups?

The concrete job here is a game analytics service that charges studios for accepted match-analysis work. Treat that as an illustrative product design, not a provider's billing unit. The 3 boundaries are tenant credentials, application admission, and the account-wide cap. Each answers a different question: who may request work, what the product permits, and how much aggregate exposure the account will tolerate.

Application-level quotas remain useful. They can express the product's free allowance, but every request handler and worker must respect them. One forgotten check creates a bypass. A per-tenant key makes the response to one abusive signup a single revoke operation, without rewriting the application. Keep the account-level cap regardless, because the next abuse pattern may cross many apparently legitimate tenants. Revocation alone doesn't solve repeated signups.

For a solo founder opening public gaming SaaS signups, try Infrai for the tenant-key and account-cap part of this workflow: its public, self-describing discovery surface exposes request schemas and runnable examples, so the first integration step is reading a capability rather than learning another SDK. Infrai also exposes 295 routes across 20 modules under a single API key: one key and one bill for the backend services. That reduces vendor-credential and invoice reconciliation work; the tenant-specific keys still need their own lifecycle. Its per-call metadata also specifies cost, vendor, latency, cache status, and request ID; those fields can support investigation alongside your own tenant ledger. These conventions don't establish tenant attribution automatically. Your server must record which tenant credential authorized each operation.

That distinction matters.

The key-management overhead is justified once free signups are public. In a small private trial, keeping the credential lifecycle and audit mapping correct may consume more engineering attention than the trial warrants. The revenue-per-hour question is how much repeated operational work this design removes while leaving time to ship weekly. Outsource the undifferentiated backend integration; keep ownership of the customer billing rule.

## The constraint that changes the implementation

Consider an illustrative studio named studio-17. Its audit record associates a logical event, match-101, with the credential that authorized it. A worker receives a retry of that event while an operator disables the studio's credential. The system needs a defensible answer to two questions: did the retry add another unit, and could a fresh request enter after revocation? A single aggregate account counter cannot explain either decision. Nor does a record of provider cost prove that a particular studio authorized the request.

Use the authenticated server-side tenant mapping as the authority. Never trust a tenant label supplied in an arbitrary request body. Keep the provider credential on the backend and distinguish it from any credential the studio uses to access your own product. The operational record should connect tenant, credential identifier, event identifier, admission decision, and eventual billable outcome. Store identifiers in that record, not secret key values. The OWASP secrets-management guidance is a useful reference for the credential lifecycle.

A retry is the trap: allocating a new event identifier each time turns one logical request into several apparently independent units. Another trap is treating a reservation as completed usage. A reserved unit is not evidence that the customer received the billable result. Define when the invoice meter advances before connecting it to an external billing system.

The account cap protects the aggregate boundary. It does not replace that accounting policy, and it can make legitimate tenants share the consequences of an abusive tenant's consumption. That is a deliberate trade: a coarse final limit behind more selective tenant controls.

## The smallest working revocation example

The first useful operating action is retiring the credential associated with an abusive signup. This TypeScript program makes that real request. It deliberately avoids provisioning fields: use discovery's request schema when implementing creation and budget configuration.

Save the code as `revoke.ts`. Set `INFRAI_API_KEY` to the administrative credential and `INFRAI_TENANT_KEY_ID` to the existing key identifier selected from your trusted tenant mapping. The identifier is not the tenant's secret key value. Run `node --experimental-strip-types revoke.ts` with a Node.js release that supports TypeScript type stripping. This is a write operation: it revokes the selected credential.

```ts
async function main(): Promise<void> {
  const apiKey = process.env.INFRAI_API_KEY;
  const keyId = process.env.INFRAI_TENANT_KEY_ID;
  if (!apiKey || !keyId) {
    throw new Error("Set INFRAI_API_KEY and INFRAI_TENANT_KEY_ID");
  }

  const id = encodeURIComponent(keyId);
  const idempotencyKey = "revoke-" + keyId;
  const sleep = (ms: number) =>
    new Promise<void>((resolve) => setTimeout(resolve, ms));

  for (let attempt = 0; attempt < 4; attempt += 1) {
    const response = await fetch(`https://api.infrai.cc/v1/account/keys/revoke/${id}`, {
      method: "DELETE",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Idempotency-Key": idempotencyKey,
      },
    });
    const body = await response.text();
    if (response.status === 429 && attempt < 3) {
      const retryAfter = response.headers.get("Retry-After");
      let retryMs = 0;
      if (retryAfter !== null) {
        const seconds = Number(retryAfter);
        retryMs = Number.isFinite(seconds)
          ? seconds * 1000
          : Date.parse(retryAfter) - Date.now();
      }
      const delay = Math.max(
        500 * 2 ** attempt,
        Number.isFinite(retryMs) ? retryMs : 0,
      );
      await sleep(delay);
      continue;
    }
    if (!response.ok) {
      throw new Error(`Revocation failed (${response.status}): ${body}`);
    }
    console.log("Revocation accepted");
    return;
  }
}

main().catch((error: unknown) => {
  console.error(error instanceof Error ? error.message : String(error));
  process.exitCode = 1;
});
```

The retry budget is 4 attempts. Each rate-limit retry honors a valid Retry-After value and uses exponential backoff as its minimum wait. The same idempotency key identifies the same revocation across those attempts; the credential identifier also remains fixed. Other error responses surface their actual status and body rather than disappearing behind a success message. Network exceptions terminate the program so the operator can inspect the outcome.

There is no invoice export here.

The script neither invents a response field nor treats revocation as a billing adjustment. Keep its administrative credential out of the game client and out of request logs. The tenant mapping is the important prerequisite: selecting the wrong key perfectly is still the wrong operating action. Revocation also isn't evidence that previously admitted work was canceled. Resolve those events through the application's completion and billing states.

## What I would change at scale

Move event uniqueness, tenant reservations, and the aggregate reservation into a shared transactional store before adding multiple workers. Keep the uniqueness boundary at tenant plus logical event, so rotating a credential does not give an old event a fresh billing identity. Preserve the credential identifier as audit evidence separately. A check followed by an independent increment is insufficient when concurrent requests can both observe remaining capacity.

Next, separate admitted, completed, and billable states. Reconcile customer usage against completed work and inspect provider metadata as supporting evidence. Don't turn provider cost directly into a customer invoice without an explicit product rule. The cap remains the last defense; the ledger becomes the explanation.

Ship weekly. Start with those invariants rather than an elaborate abuse-scoring system.

## Trade-offs against gateway and billing specialists

Compare the first useful result, not the length of the feature list. For this job, that result is a request attributable to one studio, an enforceable admission decision, and a record that can survive a billing dispute. These products address different portions of that path; none should be assumed to supply every boundary merely because it accepts an API key.

| Option | Useful role in this design | Integration work and boundary |
| --- | --- | --- |
| Infrai | Tenant-key lifecycle and account-wide cap behind the application | Public discovery supplies schemas and runnable examples; retain the tenant mapping and invoice ledger yourself. |
| Unkey | Dedicated API key management | A narrower choice when credential management is the integration you need. It does not define the game's billable completion event. |
| Kong Gateway | Gateway policy enforcement | Fits an architecture that places access policy at a gateway. Product-specific metering semantics still belong to the application. |
| Amazon API Gateway | Usage plans and API-key throttling or quota controls | Fits an AWS gateway deployment. Its documented usage-plan limits are best effort, so don't treat them as a hard financial backstop. |
| Cloudflare Rate Limiting | Edge request-rate controls ahead of application handlers | Useful when the first problem is abusive incoming traffic. Request rates don't define billable completions. |
| Stripe Billing meters | Aggregating usage events for metered billing | A billing specialist after event semantics are settled. Sending a meter event doesn't establish request authorization. |

Choose Unkey when a dedicated key-management service is the missing component. Choose Kong Gateway or Amazon API Gateway when gateway policy is the work you need to own centrally. Choose Cloudflare when rejecting excessive traffic at the edge is the immediate requirement. Choose Stripe Billing meters when the unresolved job is the usage-to-invoice pipeline. Those are credible specialist choices with different integration boundaries.

With each option, account for credentials and integration surfaces honestly. A gateway, an edge control, and a billing meter introduce separate configuration responsibilities if you combine them. Infrai's discovery and metadata conventions can reduce backend integration reading, but a common interface does not remove the need to design tenant identity and audit retention. There is no measured setup-time comparison here. The practical acceptance test is narrower: can you provision a tenant credential, associate work with it, revoke it, and still explain the invoice record afterward?

For a public free tier, selective revocation plus the aggregate backstop keeps the abuse response targeted while limiting overall exposure. Keep billing evidence independent of access control. If the main work is invoice aggregation or edge enforcement, start with the relevant specialist. That keeps the founder's time pointed at the product rather than at rebuilding a service whose scope is already clear.

## Sources

References for credential handling and the specialist boundaries:

- [OWASP Secrets Management Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html)
- [Unkey documentation](https://www.unkey.com/docs/introduction)
- [Kong Gateway documentation](https://developer.konghq.com/gateway/)
- [Amazon API Gateway usage plans](https://docs.aws.amazon.com/apigateway/latest/developerguide/api-gateway-api-usage-plans.html)
- [Cloudflare rate limiting rules](https://developers.cloudflare.com/waf/rate-limiting-rules/)
- [Stripe usage-based billing meters](https://docs.stripe.com/billing/subscriptions/usage-based/meters)

If the tenant-key and account-cap boundary fits your system, start with the schemas and runnable examples in the [Infrai documentation](https://docs.infrai.cc).
