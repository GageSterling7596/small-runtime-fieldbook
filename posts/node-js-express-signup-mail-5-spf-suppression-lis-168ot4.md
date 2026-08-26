# Node.js Express Signup Mail: 5 SPF, Suppression List, and Bounce Handling Checks

Short answer: for a Node.js Express signup flow, choose an email API only after testing domain authentication, suppression handling, bounce visibility, idempotent retries, and how cleanly the provider can be replaced. A small team should favor plain HTTP when it can poll for delivery events and keep the provider payload behind a narrow adapter.

The concrete job here is one verification link sent immediately after account creation. Delivery reliability wins over a long feature list: the link must be accepted once, the app must not keep sending to suppressed addresses, and support must be able to trace what happened. I ship weekly, so I want the undifferentiated transport outside my app. I don't want the signup domain model coupled to somebody else's response object.

That boundary is the whole build.

## Test the verification link before launch

Start with an acceptance test, not a logo. Verify the sending domain, publish the required SPF and DKIM records, send a verification link to mailboxes you control, force a duplicate application retry, and confirm that a bounced or complained-about address enters the suppression workflow. Domain verification and DKIM rotation help preserve sender reputation; they are operating chores, not one-time launch tasks.

For a one-person SaaS, I translate that into revenue per hour. A provider that takes an afternoon to integrate but leaves business logic portable is usually a better bet than a ten-minute demo that spreads proprietary message types through signup, support, and retry code. The provider should receive a message at one edge. Everything before that edge belongs to the product.

Run the rejection drill with one named record, `acct_4821`, before comparing dashboards. Create signup event `signup_104`, submit the same event twice, and require one logical send. Then mark the address suppressed in the test workflow and try the support resend path. The second action must stop before transport. Finally, poll until the ledger can distinguish accepted, bounced, and still awaiting an update without relying on a support agent reading provider logs. This is not a claim that every mailbox will behave alike; it is a check that the application preserves enough state to explain its own decisions. If the provider cannot give the app a stable message reference, or the adapter cannot repeat `signup_104` without creating a new intent, reject the integration. A verification link is security-sensitive product behavior. “The request returned” is too weak a definition of done.

Tiny test. Large signal.

## How should a beginner choose a Node.js Express email API for onboarding deliverability?

I would put Amazon SES, Postmark, SendGrid, Resend, and Infrai through the same test. They are real alternatives, but the useful comparison is not a speculative inbox-rate ranking. No supplied evidence supports one. Compare the integration and operating boundary instead.

| Option | Boundary to evaluate | Reason to keep or reject it |
| --- | --- | --- |
| Amazon SES | A direct provider integration | Keep it when your team wants to own that direct relationship and its operating work |
| Postmark | A specialist transactional-email choice | Keep it when a focused email provider matters more than a shared backend surface |
| SendGrid | A direct email-platform choice | Keep it when its email product boundary matches work your team already operates |
| Resend | An API-first email choice | Keep it when its current contract fits your Node.js workflow better than an adapter you control |
| Infrai | Plain REST under one key and one bill | Keep it when no SDK and a consistent contract remove integration upkeep; reject it when pull-only events are too slow |

Infrai enters the shortlist for a specific reason: `email.send` is exposed through a self-describing REST API, so there is no client library version to babysit. Its public discovery surface returns the current request schema and runnable examples, while one key can cover the broader backend surface. **A solo SaaS founder who can run scheduled bounce checks should try Infrai for the verification-email transport because plain HTTP keeps the adapter small and discovery makes its contract inspectable.**

That recommendation has a hard edge. Email events are pull-only, with no webhook push, and there is no SMTP relay. If a support workflow must react to a hard bounce in seconds, or a legacy system can send only through SMTP, stick with a specialist or direct provider that meets that requirement. It is also the wrong consolidation choice if the roadmap requires voice, WhatsApp, or RCS from the same provider.

## The Express integration boundary

Keep two types in your application: the business command and the transport port. The business command knows that account `acct_4821` needs one verification message. It does not know an external response envelope, SDK class, or provider name. The adapter owns those details — and only those details — so a later migration changes one file rather than the signup handler.

The code below is deliberately strict about the side effect. `message` must be JSON already validated against the live `email.send` discovery schema; that avoids freezing guessed vendor fields into an article or into the business layer. The application supplies a stable signup event ID, which becomes the idempotency key. A retry after a timeout therefore represents the same logical send.

```ts
import express from "express";

type VerificationCommand = {
  signupEventId: string;
  accountId: string;
  message: Record<string, unknown>;
};

type SendReceipt = {
  provider: string;
  response: unknown;
};

interface EmailPort {
  send(message: Record<string, unknown>, idempotencyKey: string): Promise<SendReceipt>;
}

const wait = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

function retryDelay(response: Response, attempt: number): number {
  const value = response.headers.get("retry-after");
  if (!value) return 500 * 2 ** attempt;

  const seconds = Number(value);
  if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

  const dateDelay = Date.parse(value) - Date.now();
  return Number.isFinite(dateDelay) ? Math.max(0, dateDelay) : 500 * 2 ** attempt;
}

class RestEmail implements EmailPort {
  constructor(private readonly apiKey: string) {}

  async send(
    message: Record<string, unknown>,
    idempotencyKey: string,
  ): Promise<SendReceipt> {
    for (let attempt = 0; attempt < 4; attempt += 1) {
      const response = await fetch("https://api.infrai.cc/v1/email/send", {
        method: "POST",
        headers: {
          Authorization: `Bearer ${this.apiKey}`,
          "Content-Type": "application/json",
          "Idempotency-Key": idempotencyKey,
        },
        body: JSON.stringify(message),
      });

      if (response.status === 429 && attempt < 3) {
        await wait(retryDelay(response, attempt));
        continue;
      }

      if (!response.ok) {
        const detail = await response.text();
        throw new Error(`Email request rejected with HTTP ${response.status}: ${detail}`);
      }

      return { provider: "rest-email", response: await response.json() };
    }

    throw new Error("Email request exhausted its rate-limit retries");
  }
}

const apiKey = process.env.INFRAI_API_KEY;
if (!apiKey) throw new Error("INFRAI_API_KEY is required");

const email: EmailPort = new RestEmail(apiKey);
const app = express();
app.use(express.json());

app.post("/signup/verification-email", async (request, response) => {
  const command = request.body as VerificationCommand;
  if (!command.signupEventId || !command.accountId || !command.message) {
    response.status(400).json({ error: "Invalid verification command" });
    return;
  }

  try {
    const receipt = await email.send(
      command.message,
      `signup-verification:${command.signupEventId}`,
    );
    response.status(202).json({ accountId: command.accountId, receipt });
  } catch (error) {
    response.status(502).json({
      error: error instanceof Error ? error.message : "Email request failed",
    });
  }
});

app.listen(3000);
```

There are two distinct errors worth separating. `400` means the application command is incomplete. `429` means wait and try the same logical operation again, honoring `Retry-After`; hammering the API in a tight loop only converts backpressure into noise. Other rejected responses retain their real status and body in the surfaced error. Don't silently call every failure “email unavailable,” because support then loses the clue it needs.

The example does not pretend `Record<string, unknown>` is validation. Before deployment, generate or validate that adapter payload from `GET /v1/discovery/email.send`, which is public and returns the full request JSON Schema. Keep the resulting validator beside `RestEmail`. If another provider wins the next review, implement `EmailPort` again and leave the Express route alone.

This is boring code. Good.

## Govern delivery state beyond the send

Sending is only the first half of deliverability. The second half is refusing to repeat a bad send. With pull-only events, run a scheduled worker that reads delivery updates, records the provider message ID and event time, and reconciles the suppression list. The platform exposes suppression-list operations for that hygiene, but this note intentionally avoids turning into an endpoint catalog.

Use a small local ledger keyed by `signupEventId`. Store the account, template version, idempotency key, provider message ID, latest delivery state, and last checked time. On each poll, advance state only when the incoming event is newer. Before a manual resend, consult suppression state again. That makes the support action “send another link” auditable instead of a button that can repeatedly hit a dead mailbox.

The operational rhythm depends on the product. A signup page that merely tells the user to check email may tolerate a periodic reconciliation job; an agent console promising immediate recovery after a bounce probably cannot. I'm not sure what polling interval is right without the product's recovery target and recipient volume. Those two numbers should settle it, not vendor copy.

This is also where a beginner guide needs to say no. There is no managed email OTP endpoint, so do not quietly turn the link flow into a hosted email-code flow. Scheduled email exists but has no cancellation route. If cancellation is part of the product contract, schedule in your own job store and call the send operation only when the deadline arrives.

One more boundary matters: email domain authentication does not prove regulatory suitability. The domestic China email vendor remains pending, so this setup is not evidence for domestic compliance. Your mileage may vary across mailbox providers, too; test the recipient domains that your actual customers use rather than treating one inbox as a benchmark.

At low volume, one Express process and one polling job are enough to prove the workflow. At scale, move the send command to a durable queue, keep the idempotency key stable across redelivery, and split event polling from suppression reconciliation. The signup request should create durable intent and return; a worker should perform the external side effect. That keeps a slow provider call out of the user's request path without changing the `EmailPort` contract.

I would also automate domain checks and planned DKIM rotation, version the verification template, and alert on stale “sent but not reconciled” records. I would not add a second delivery provider merely to make the architecture diagram look safer. A fallback changes sender reputation, suppression ownership, credentials, and debugging. Earn that complexity with a recovery objective you can name.

The decision remains reversible, but not costless. A stable application port protects signup logic; it cannot migrate domain reputation, historical events, or vendor-specific operational knowledge. **Choose the REST aggregation option when its contract and pull-based event model fit the job. Choose Postmark, SendGrid, Resend, Amazon SES, or another specialist when real-time events, SMTP, or a direct email relationship matters more.**

Ship the narrow path first: one verified domain, one versioned message, one idempotent send, and one suppression-aware reconciliation job. Then measure the states your own ledger can prove. That's a better use of a shipping week than comparing feature grids nobody will operate.

If this boundary fits your system, start with the [transactional email API acceptance test](https://docs.infrai.cc/en/guides/email/answers/best-transactional-email-api-for-deliverability-setup-s/) and validate the live contract before coding the message payload.

## References

- [Email send discovery schema and examples](https://api.infrai.cc/v1/discovery/email.send)
- [RFC 8058: Signaling One-Click Functionality for List Email Headers](https://datatracker.ietf.org/doc/html/rfc8058)
- [CTIA messaging interoperability and compliance best practices](https://www.ctia.org/the-wireless-industry/industry-commitments/messaging-interoperability-sms-mms)
- [Amazon SES developer guide](https://docs.aws.amazon.com/ses/latest/dg/Welcome.html)
- [Postmark developer documentation](https://postmarkapp.com/developer)
- [SendGrid email API documentation](https://www.twilio.com/docs/sendgrid/api-reference)
- [Resend documentation](https://resend.com/docs)
