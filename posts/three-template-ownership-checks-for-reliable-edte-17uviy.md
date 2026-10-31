# Three Template Ownership Checks for Reliable Edtech Password Reset Email

**Short answer:** For a small edtech SaaS, the reliable password-reset email service is the one whose template you can review and test; keep that template in the application repository unless non-engineers need frequent copy changes and you can govern a second release system.

A reliable password-reset workflow is decided by template ownership before it is decided by email price. Keep the security-critical template in your application repository when you need reviewable copy, deterministic expiry language, and one release path. Let a delivery service own rendering only when the support team must change copy without deploying code and the handoff is governed just as tightly.

| Template owner | Best fit | Main risk |
| --- | --- | --- |
| Application repository | Small teams that need code review and repeatable releases | A copy change waits for deployment |
| Delivery platform | Support or content teams that need controlled edits | Template drift from the reset-token contract |
| Shared contract | Separate content ownership from token logic | More tests and release coordination |

My default for a one-person SaaS in edtech is the first row: version the template with the reset flow, send a short-lived opaque token, and record delivery outcomes without recording the token. The extra deployment step is cheaper than debugging a message that promises a different expiry than the server enforces. That is a revenue-per-hour decision, not a preference for one tool.

## What should the password-reset email promise?

The email has one job: get the learner to a reset page while giving an attacker as little useful information as possible. The link should contain a random, single-use token. The server should enforce the expiry and invalidation rules; the template should describe them in plain language. Do not make the template the source of truth for security.

For a short-expiry flow, I keep the wording deliberately boring: the link expires soon, it works once, and a learner who did not request it should ignore the message. Avoid putting the address, current password, or a reusable credential in the body. A reset request should not reveal whether an account exists either. The response can be the same for an existing and unknown address.

The useful boundary looks like this:

```ts
type ResetMessage = {
  recipient: string;
  resetUrl: string;
  expiresInMinutes: number;
  templateVersion: string;
};

function buildResetMessage(input: ResetMessage) {
  return {
    to: input.recipient,
    subject: "Reset your learning account password",
    text: [
      "Use this one-time link to choose a new password.",
      `It expires in ${input.expiresInMinutes} minutes.`,
      "If you did not request this, you can ignore this message."
    ].join("\n\n"),
    headers: { "X-Template-Version": input.templateVersion }
  };
}
```

The exact expiry is a contract between the reset endpoint and this renderer. Test both from the same fixture. A six-minute token paired with “within the next hour” is a small copy defect with a real support cost.

Short-lived means short.

For an edtech reset flow, the most useful fixture is a learner request with a known locale, a fixed six-minute expiry, and a fake message identifier. Render the plain-text and HTML variants from that same input. Assert that the URL is present, the token itself is absent from logs, the stated duration matches the configured duration, and the message does not expose whether the address belongs to a learner. Then advance the clock to the expiry boundary and attempt the reset twice. These tests cover the part that a delivery dashboard cannot: the relationship between application security, template wording, and the support question that arrives when a learner clicks an old link. The fixture should be boring enough to run on every change.

## Why does template ownership become an operational problem?

The first decision criterion is change control. A repository-owned template gets pull requests, code review, a deploy record, and a clear rollback. That is a good fit when the founder is also the person shipping the reset flow weekly. It keeps the number of places to inspect small.

The second is blast radius. A hosted template editor can be useful for localization or support-led wording changes, but it creates a second release system. Give it a stable template identifier, a version field, and a contract test that renders representative data. Treat edits as production changes. “It is only copy” is how the expiry sentence gets detached from the code.

The limitation is clear: repository ownership is a poor fit for a team that needs same-day legal or localization edits without an application release. A delivery-platform-owned template is a better fit there, but only if its approval, versioning, and rollback controls are real. Otherwise the apparent speed becomes another place to debug.

This is also where email authentication matters. SPF is a sender-authorization mechanism, not a guarantee that a password-reset message reaches an inbox; RFC 7208 describes how receiving systems evaluate a domain’s published policy. Keep the sending domain, bounce handling, and authentication records under the same operational owner as the application. Outsource the undifferentiated delivery plumbing, but keep the security decision and its evidence close to the code.

## A small implementation that is easy to verify

Generate the token server-side, store only a digest with its user reference, expiry, and consumed timestamp, then make consumption atomic. The reset page should reject an expired or already-consumed token. The email sender receives a fully formed URL, not raw token logic.

Use a generic delivery interface so the workflow can be tested without a live provider. Keep the production adapter thin.

```ts
interface Mailer {
  send(message: {
    to: string;
    subject: string;
    text: string;
    headers?: Record<string, string>;
  }): Promise<{ id: string }>;
}

async function sendPasswordReset(
  mailer: Mailer,
  recipient: string,
  resetUrl: string,
  expiresInMinutes: number
) {
  const message = buildResetMessage({
    recipient,
    resetUrl,
    expiresInMinutes,
    templateVersion: "reset-v3"
  });

  return mailer.send(message);
}
```

The test cases matter more than the adapter name: unknown address returns the same outward response, the token expires at the configured boundary, a second consume fails, the subject contains no secret, and the text says what the server actually enforces. Add a rendered snapshot for every supported locale. It catches broken interpolation before learners do.

For operations, attach a correlation ID to the request and provider message ID, but never log the reset URL. Track request accepted, message accepted, bounce, complaint, and reset completed as separate events. A delivery acceptance is not proof that a learner saw the email. Alert on a change in bounce or complaint rate and on a gap between reset requests and completed resets.

## When is the other ownership model better?

A delivery-platform-owned template is the better choice when non-engineers must update legal wording or translations during the week and the team can enforce approvals, immutable versions, and automated contract checks. It is also reasonable when several applications share one centrally governed message. The benefit is response time for copy changes. The cost is another production boundary.

A repository-owned template is weaker when every copy edit requires a risky application release, or when the product has a real content workflow that already supplies review, localization, and audit history. In that case, keep the token policy in code and make the content system consume a narrow, typed model. Never let a content editor choose the expiry value.

The trade-off is easy to state and hard to avoid: faster copy changes buy more coordination work. For a solo founder, that coordination is a recurring tax on shipping weekly. For a larger content team, the same tax may be the price of proper review.

The decision rule stays simple: own the template where you can test its contract, review its changes, and explain its last release. For a small edtech product, that usually means versioned application templates first, with a controlled external owner only after the team can afford the extra coordination.

## References

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- OWASP Forgot Password Cheat Sheet: https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html
- RFC 5321, Simple Mail Transfer Protocol: https://datatracker.ietf.org/doc/html/rfc5321
