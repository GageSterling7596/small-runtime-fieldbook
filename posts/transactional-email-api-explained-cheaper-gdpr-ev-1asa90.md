# Transactional Email API Explained: Cheaper GDPR Evidence for Europe

The phrase “cheaper Resend alternative” is an incomplete comparison for a transactional email API. For a game account in Europe, a password reset is useful only if it arrives before the token expires and leaves enough records to investigate access requests, complaints, and delivery failures.

**TL;DR:** Put a small provider-neutral adapter behind your game account service. Require authenticated custom-domain sending, documented processing terms, controllable retention, event export, and a stable message identifier. Keep the reset token short-lived and single-use, but retain a separate, token-free audit record under a written retention policy. The least expensive acceptable option is the one that passes those checks without creating recurring evidence work.

| Choice | Compliance evidence | Switching effort | Operational load | Best fit |
| --- | --- | --- | --- | --- |
| Direct email API | Depends on export and retention controls | Medium | Low | A solo team that can verify the processor and event trail |
| Portable adapter across APIs | Consistent internal record; provider detail still varies | Low after setup | Medium | A game likely to change regions or processors |
| Self-operated mail transfer agent | Full control, plus full responsibility | High | High | Teams with mail and abuse specialists |

For a one-person SaaS, I would start with the portable adapter and one qualified processor. It costs a little integration time, but it outsources undifferentiated mail transport while keeping the evidence schema under my control. That is the trade I can defend while shipping weekly.

## What should a cheaper transactional email API alternative prove?

GDPR accountability changes the comparison. The controller must be able to demonstrate compliance, and a processor relationship needs documented instructions and appropriate safeguards. A vague claim that a service is “GDPR compliant” does not answer where data is processed, which subprocessors receive it, how long event data remains available, or how deletion requests propagate.

Write an evidence checklist before opening a pricing page. Ask for the data processing agreement, processing locations, subprocessor list and change process, security measures, deletion procedure, breach-notification terms, and a usable export of delivery events. Confirm which fields appear in logs. A reset URL is a credential; it should not be copied into an event payload, template variable log, analytics tag, or support transcript. The internal record can be small: account pseudonym, message purpose, template revision, provider message ID, acceptance time, terminal delivery state, and expiry time. Do not store the raw reset token or the full reset URL. Keep the mapping between an account and a provider message ID only as long as the documented support, security, and legal purposes require it. Retention is not one number. A token may expire in 15 minutes, transient delivery diagnostics may need a short operational window, and an aggregated security record may have a different justified lifetime. Define each clock, its purpose, and its deletion job. Then test deletion. The same evidence model can cover a welcome email, but its purpose and retention decision should remain distinct from account recovery; combining both merely because they share an API makes later review harder.

Keep it narrow.

## Can the custom domain prove who sent the message?

A custom From address is cosmetic until the domain is authenticated. SPF lets a domain publish which hosts may use it in SMTP identity checks. DKIM attaches a domain-linked signature to the message. DMARC builds on aligned identifiers and publishes a policy plus reporting instructions. Together they make authorization and alignment inspectable; they do not guarantee inbox placement.

Use a dedicated transactional subdomain so account mail has a clear boundary from marketing traffic. Publish only the senders that actually need authorization. RFC 7208 limits the number of DNS-querying terms evaluated during an SPF check to 10, so repeatedly including services without removing old ones can produce a permanent error instead of stronger authentication.

Roll out deliberately. Verify SPF and DKIM first, observe DMARC reports, correct legitimate alignment failures, and only then tighten policy. Also test the visible From domain, return path, and DKIM signing domain as received by a mailbox. A green dashboard is weaker evidence than the headers on a real message.

This catches a common trap: domain setup can pass once and silently decay after a DNS migration. Schedule a DNS and received-header check. Weekly is cheap compared with finding the problem during an account takeover report.

## The two criteria that decide the shortlist

The first criterion is **exportable, minimizable evidence**. A service should return a stable identifier at submission and expose state changes such as accepted, delivered, delayed, bounced, or complained. You need a way to authenticate event delivery, reject duplicates, tolerate reordering, and export or delete associated personal data. Exact event names matter less than whether your adapter can map them into a small internal state machine.

The second is **processor governance**. Read the contract and subprocessor material, not the region selector alone. A European endpoint does not establish the full data path. Support access, telemetry, backups, and subprocessors can matter too. Record the review date and the documents reviewed so a future change is visible rather than tribal knowledge.

Cost comes after both gates. Compare the workload at your real volume, including retries, retained events, support needs, and the engineering hours required to produce an audit trail. For a solo operator, saving a small amount on sends can be a bad exchange if every data request becomes manual archaeology. Revenue per hour is the useful lens. A cheaper alternative is not suitable when it cannot produce the required processing documents or export the events your incident review needs.

Run a fixed trial against each candidate. Send reset mail to accounts you control at several major mailbox operators, force a hard bounce with a reserved invalid domain, submit the same event twice, rotate a webhook secret, and request deletion of the test recipient. Capture the contract answers and test results in the repository. No invented score. Pass or fail each requirement.

No document, no pass.

## A minimal evidence-first implementation

The application should create the reset grant, persist a token-free audit row, and hand a narrowly scoped message to an adapter. The provider response supplies correlation, not business truth. Delivery events update the audit record later.

```ts
import { createHash, randomBytes } from "node:crypto";

type ResetMessage = {
  recipient: string;
  resetUrl: string;
  expiresAt: Date;
  idempotencyKey: string;
};

type Submission = {
  messageId: string;
  acceptedAt: Date;
};

interface TransactionalMailer {
  sendPasswordReset(message: ResetMessage): Promise<Submission>;
}

type ResetAudit = {
  accountRef: string;
  purpose: "password-reset";
  tokenDigest: string;
  expiresAt: Date;
  providerMessageId?: string;
  submittedAt?: Date;
  state: "created" | "accepted" | "failed";
};

async function requestPasswordReset(
  accountId: string,
  recipient: string,
  mailer: TransactionalMailer,
): Promise<void> {
  const token = randomBytes(32).toString("base64url");
  const tokenDigest = createHash("sha256").update(token).digest("hex");
  const expiresAt = new Date(Date.now() + 15 * 60 * 1000);
  const accountRef = createHash("sha256").update(accountId).digest("hex");

  const audit: ResetAudit = {
    accountRef,
    purpose: "password-reset",
    tokenDigest,
    expiresAt,
    state: "created",
  };
  await saveAudit(audit);

  try {
    const submission = await mailer.sendPasswordReset({
      recipient,
      resetUrl: `https://accounts.example/reset?token=${encodeURIComponent(token)}`,
      expiresAt,
      idempotencyKey: tokenDigest,
    });
    await markAccepted(tokenDigest, submission.messageId, submission.acceptedAt);
  } catch (error) {
    await markFailed(tokenDigest);
    throw error;
  }
}
```

The sample deliberately omits database implementations and provider payloads. `saveAudit`, `markAccepted`, and `markFailed` must be atomic enough for your retry model, and the adapter must never log `resetUrl`. Store the token digest so redemption can compare a presented token without retaining the credential itself. On successful redemption, invalidate the grant immediately and revoke any sibling reset grants according to the account security policy.

Treat timeouts as unknown outcomes. A timed-out request may already have been accepted, so retry with the same idempotency key where the chosen service supports that contract; otherwise reconcile by message metadata before sending again. The recipient should see one useful reset message, not a burst of competing links.

Webhook ingestion needs the same restraint. Verify the signature over the raw request body according to the processor's documented scheme, record an event ID, and make updates idempotent. Do not grant or deny account access from an email “delivered” event. SMTP delivery only describes transport progress.

## When is the runner-up architecture better?

A direct integration is reasonable when the processor already meets every evidence requirement, the application is unlikely to change jurisdictions, and the team values fewer moving parts. Keep the boundary narrow anyway. Provider response objects spread quickly through a codebase, and that makes the next contract review an application rewrite.

Self-operation becomes plausible when policy demands infrastructure and logs under direct control, or when specialized deliverability staff already exist. It is rarely a shortcut. Queue durability, reputation, feedback loops, abuse handling, DNS, key rotation, suppression, monitoring, and incident response become your work.

The portable adapter remains the practical default for a small game service because it isolates change without pretending providers are interchangeable. Its downside is extra code, mapping tests, and an internal schema that someone must maintain. It is not a fit when one processor-specific capability is central to the application or when the team cannot own the adapter's event reconciliation. Contracts, event semantics, retention, and delivery behavior still differ. Normalize only what the account workflow actually consumes.

Ship the narrow path. Review the processor evidence on a schedule, test authentication from the receiving side, and rehearse export and deletion before a real request arrives. That produces a defensible password-reset system without turning email infrastructure into the product.

## References

- [Regulation (EU) 2016/679 (GDPR), Articles 5, 28, and 32](https://eur-lex.europa.eu/eli/reg/2016/679/oj)
- [RFC 7208: Sender Policy Framework (SPF)](https://datatracker.ietf.org/doc/html/rfc7208)
- [RFC 6376: DomainKeys Identified Mail (DKIM) Signatures](https://datatracker.ietf.org/doc/html/rfc6376)
- [RFC 7489: Domain-based Message Authentication, Reporting, and Conformance (DMARC)](https://datatracker.ietf.org/doc/html/rfc7489)
- [OWASP Forgot Password Cheat Sheet](https://cheatsheetseries.owasp.org/cheatsheets/Forgot_Password_Cheat_Sheet.html)
- [Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
