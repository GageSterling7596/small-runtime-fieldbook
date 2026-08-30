# Owning Verification Templates: SMS and Email OTP Security for SaaS Login

Short answer: for a logistics SaaS sending a signup verification link in the US and EU, start with email when your team needs full ownership of the message template, keep SMS as an explicit fallback, and apply one rate-limit budget across both channels.

| Decision | Email-first | SMS-first |
| --- | --- | --- |
| Template ownership | Full layout, explanatory copy, and branded destination context | Short copy shaped by a tight message surface |
| Signup fit | Good when the address is already the account identifier | Good when the phone number is already operationally essential |
| Fallback | Offer SMS after a deliberate user action | Offer email after a deliberate user action |
| Security posture | Single-use link, short expiry, server-side redemption | Single-use code or link, short expiry, server-side redemption |
| Operational burden | Domain authentication and inbox placement | Regional routing, phone normalization, and message-length discipline |

The recommendation follows the job. A new dispatcher is verifying an account, not responding to an urgent delivery exception. Email gives the product enough room to identify the workspace, explain why the message arrived, and show the destination before the user clicks. SMS remains valuable, but making it an automatic parallel send doubles the abuse surface and makes the fallback impossible to reason about.

The catch is simple: email-first is not suitable when a verified phone number is the actual account anchor or when users routinely sign up away from an inbox. In that case, choose SMS first and retain email as recovery. The channel should follow the identifier and workflow, not a generic claim that one medium always delivers better.

## What should a US and EU SaaS compare for SMS OTP vs email OTP?

Start with template ownership, because it changes what the verification message can safely communicate. A logistics account may belong to a carrier, a warehouse, or a shipper with a similar name. In email, the template can include the workspace name, the requested action, a plain-language expiry statement, and a visible application domain. That context helps a recipient decide whether the request makes sense before opening the link.

SMS has a different strength: it keeps the action close to a phone number that may already be part of dispatch operations. But less space means every word has to earn its place. The template should identify the application, state the action, and avoid stuffing operational details into the message. The landing page can provide the fuller context after it validates the token.

Ownership also includes versioning. Store email and SMS templates beside application code, review copy changes, and bind every deployed template to a stable event such as `signup_verification_requested`. The delivery adapter should receive rendered content; it shouldn't contain business copy. This keeps a provider change from becoming a rewrite of the signup flow.

Don't let fallback become a second template system. Both channels should express the same event, expiry policy, support route, locale, and workspace identity. Only the presentation changes. If legal or support copy differs by region, make that an explicit template input and test it, rather than branching deep inside a delivery client.

For email, domain policy is part of ownership too. DMARC defines a way for a domain owner to publish handling policy and receive reports about authentication results. That does not make inbox placement automatic, but it gives the sending domain an explicit authentication policy boundary. Treat DNS policy, the visible From domain, and the links inside the template as one release surface, not three unrelated chores.

## Template ownership is an operational boundary

The useful boundary is small: the application creates a verification challenge, the template layer renders it, and a channel adapter transports it. Redemption stays in the application. A delivery provider never decides whether a token is valid, how many attempts remain, or which workspace the user may join.

That separation pays off every week. Copy can change without touching token semantics. A transport can change without changing template variables. Most important for a solo SaaS, a failed send does not turn into an ambiguous authentication state: the challenge remains pending until redemption or expiry, while delivery attempts are recorded separately.

Use a narrow template contract. `workspaceName`, `verificationUrl`, `expiresAt`, `locale`, and a support address are usually enough for this signup job. Do not pass a whole user record into a renderer. A broad object is convenient for one sprint and expensive forever, because every template can quietly begin depending on unrelated personal data.

Keep the logs narrow as well. Record the challenge ID, channel, template version, attempt number, normalized destination fingerprint, and delivery state. Avoid logging the raw token or complete verification URL. The revenue-per-hour lens matters here: searchable, privacy-conscious events shorten support work without creating a second database of credentials.

Email-open telemetry deserves special skepticism. Apple documents that Mail Privacy Protection can prevent senders from learning whether a recipient opened a message and can mask the recipient's IP address. An “open” is therefore not a dependable verification signal. The application should measure accepted sends, user-requested resends, successful redemptions, expiry, and channel switches instead. Redemption is the event that matters.

## Delivery, security, and cost belong in one decision

Deliverability is not a single percentage that can settle the channel choice. It is a sequence: the application accepts a request, creates a challenge, hands a message to a transport, and later observes redemption or expiry. Each boundary needs its own state. If the UI collapses all of them into “sent,” support will have no way to distinguish a typo from a delayed message or an abandoned signup. Security starts before transport. Generate an opaque, single-use secret; store only a verifier representation; bind it to the intended account action; expire it; and invalidate it after successful redemption. A second request should not silently make the first challenge easier to guess. It should either replace the active challenge under a documented policy or reuse the existing challenge while respecting the resend window. Rate limiting must span channels. Otherwise a user who reaches the email ceiling can switch to SMS and reset the counter. Use separate controls for challenge creation, delivery attempts, and token guesses, then aggregate them around more than one dimension: account, destination, network signal, and time window. Exact limits are product decisions. I'm not sure which values fit your traffic until you have clean redemption and abuse telemetry; your mileage may vary, especially when warehouse networks put many legitimate users behind one egress address. Cost belongs in the model, but it shouldn't lead it. Track cost per requested verification, per redeemed verification, and per fallback redemption. The last measure catches a costly pattern: a primary channel that looks inexpensive per send but pushes many users into a second message. Put a budget alarm around the whole verification event, not around one adapter's invoice. One more constraint matters: never send both channels immediately just to improve the odds. It spends twice, teaches users to expect duplicate credentials, and gives an attacker two redemption surfaces. Make fallback a visible action after a resend delay, charge it to the same rate-limit budget, and invalidate competing challenges when one succeeds.

Ship the first policy with boring defaults, then review the funnel weekly. That's enough.

Watch redemption.

## A focused TypeScript implementation

The following example keeps policy in the application and transport details behind an interface. Its limits are illustrative configuration, not universal best practice. The important parts are shared budgeting, server-side redemption, stable template inputs, and explicit fallback.

```ts
type Channel = "email" | "sms";

type VerificationRequest = {
  accountId: string;
  workspaceName: string;
  email: string;
  phone?: string;
  locale: "en-US" | "en-GB" | "de-DE";
};

type Delivery = {
  send(input: {
    channel: Channel;
    destination: string;
    template: "signup-verification";
    variables: {
      workspaceName: string;
      verificationUrl: string;
      expiresAt: string;
      locale: VerificationRequest["locale"];
    };
  }): Promise<{ messageId: string }>;
};

type ChallengeStore = {
  create(input: {
    accountId: string;
    verifierHash: string;
    expiresAt: Date;
  }): Promise<{ id: string }>;
  recordDelivery(input: {
    challengeId: string;
    channel: Channel;
    messageId: string;
  }): Promise<void>;
};

type Budget = {
  consume(key: string, action: "create" | "deliver" | "guess"): Promise<void>;
};

declare const delivery: Delivery;
declare const challenges: ChallengeStore;
declare const budget: Budget;

async function requestSignupVerification(
  request: VerificationRequest,
  channel: Channel = "email",
): Promise<{ challengeId: string }> {
  const destination = channel === "email" ? request.email : request.phone;
  if (!destination) throw new Error("CHANNEL_DESTINATION_MISSING");

  // Both channels consume the same account-level delivery budget.
  await budget.consume(`signup:${request.accountId}`, "create");
  await budget.consume(`signup:${request.accountId}`, "deliver");

  const secret = crypto.randomUUID();
  const verifierHash = await hashSecret(secret);
  const expiresAt = new Date(Date.now() + 10 * 60 * 1000);
  const challenge = await challenges.create({
    accountId: request.accountId,
    verifierHash,
    expiresAt,
  });

  const verificationUrl = new URL("https://app.example.test/verify");
  verificationUrl.searchParams.set("challenge", challenge.id);
  verificationUrl.searchParams.set("secret", secret);

  const sent = await delivery.send({
    channel,
    destination,
    template: "signup-verification",
    variables: {
      workspaceName: request.workspaceName,
      verificationUrl: verificationUrl.toString(),
      expiresAt: expiresAt.toISOString(),
      locale: request.locale,
    },
  });

  await challenges.recordDelivery({
    challengeId: challenge.id,
    channel,
    messageId: sent.messageId,
  });

  return { challengeId: challenge.id };
}
```

In production, `hashSecret` should be a reviewed cryptographic implementation, and the redemption handler should compare the submitted secret to the stored verifier, consume a guess budget, check purpose and expiry, and atomically mark the challenge used. Those details stay outside `Delivery` on purpose. Transport is undifferentiated work; authentication policy is product risk.

The error names are also intentional. `CHANNEL_DESTINATION_MISSING` is safe for internal handling, while the public response should avoid revealing whether an account exists. A rate-limit rejection can map to an application-defined `OTP_RATE_LIMITED` response and a retry time without exposing which key triggered the rule.

Test this as a state machine rather than a collection of happy-path mocks. Cover initial request, resend inside the waiting window, fallback to the alternate channel, wrong secret, expired secret, successful redemption, and a second redemption attempt. Then test template snapshots per locale. A template edit can break a link just as easily as an adapter edit can.

## When is the runner-up channel the better primary?

Stick with SMS first when the phone number is the durable account identifier, the user is already working from a mobile device, and the template only needs a short verification instruction. This is common when a driver or field operator joins an existing logistics workspace. Email can remain the recovery route for users who have an address on file.

Choose email first when workspace context reduces mistaken clicks, when localization needs more room, or when the inbox address is the login identifier. That fits an administrator creating a carrier account from a desktop. SMS then works as an intentional fallback after the user confirms the masked number.

Neither channel is a complete recovery plan. If losing one inbox or phone number can permanently lock out the only workspace owner, add a separately designed recovery path with stronger review than ordinary OTP fallback. Don't disguise recovery as another resend button.

The final rule is mundane: own the challenge and the templates, choose the primary channel that matches the account identifier, and measure redemption rather than opens. Outsource transport. Keep policy close. Then ship the next feature.

## References

- https://datatracker.ietf.org/doc/html/rfc7489
- https://support.apple.com/guide/iphone/use-mail-privacy-protection-iphf084865c7/ios
