# Designing an Idempotent Webhook Backoff Policy (When the Spend Ceiling Wins)

A prepaid balance can run out while nobody is watching. That makes a webhook retry policy a revenue control, not housekeeping: backoff protects delivery, but giving up too early loses the signal and retrying forever can violate the same spend ceiling the signal is meant to protect.

Short answer: register an explicit retry policy, make the Express consumer idempotent, inspect delivery history, and define an observable terminal state after the final attempt.

My decision rule is blunt. Retry transient delivery failures within a fixed attempt and time budget. Stop when the next attempt would cross the operational ceiling, then route the exhausted delivery to a queue or an alert that someone actually owns. Don't let it disappear.

## What constraint changes the webhook retry decision?

The usual advice is to maximize eventual delivery. A prepaid developer tool has a second objective: refuse as little legitimate traffic as possible without allowing unattended usage to run past its funded balance. Those goals pull in opposite directions. A short retry window can miss a temporary receiver outage; a long, aggressive window can keep spending resources after the notification has lost its chance to prevent refused traffic.

So I start with exposure, not a fashionable backoff sequence. Suppose the balance is $120, the internal intervention floor is $30, and the workload can consume $2 per minute. Those are example assumptions, not vendor limits. They leave 45 minutes of operating room. The retry policy, escalation delay, and human response time must fit inside that room. If they don't, changing exponential backoff from one curve to another is beside the point.

I'm not sure which curve is right before delivery history exists; nobody has the workload evidence yet. Start conservatively, then use actual delivery records to see how many failures recover on the second or third attempt and how long recovery takes. A `429` should wait, and a receiver should honor `Retry-After` when it makes outbound follow-up calls. Authentication or schema failures need correction, not a high-frequency retry loop.

The last attempt is a product decision. After it, preserve the event and expose the failure. Silent give-up is how a customer becomes the monitoring system.

## How should a Node.js Express consumer handle webhook retry policy and backoff?

Assume the ingress layer normalizes each provider payload into the tiny contract below: a stable event ID and the latest balance in cents. The HMAC header is part of this example receiver's contract; it is not a claim about any vendor's webhook signature format. In production, use the sender's documented verification scheme before touching the idempotency store.

The important boundary is the database transaction. Recording the event and updating the balance happen together. If the process commits and crashes before returning `200`, the sender can retry; the second request sees the event ID and returns success without applying the balance update again.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import { DatabaseSync } from "node:sqlite";
import express, { Request, Response } from "express";

const secret = process.env.WEBHOOK_SECRET;
if (!secret) throw new Error("WEBHOOK_SECRET is required");

const db = new DatabaseSync("balance-events.db");
db.exec(`
  CREATE TABLE IF NOT EXISTS processed_events (
    event_id TEXT PRIMARY KEY,
    processed_at TEXT NOT NULL
  );
  CREATE TABLE IF NOT EXISTS account_state (
    account_id TEXT PRIMARY KEY,
    balance_cents INTEGER NOT NULL
  );
`);

type BalanceEvent = {
  eventId: string;
  accountId: string;
  balanceCents: number;
};

function hasValidSignature(raw: Buffer, supplied: string): boolean {
  const expected = createHmac("sha256", secret).update(raw).digest("hex");
  const left = Buffer.from(expected, "utf8");
  const right = Buffer.from(supplied, "utf8");
  return left.length === right.length && timingSafeEqual(left, right);
}

const app = express();
app.post(
  "/webhooks/balance",
  express.raw({ type: "application/json", limit: "64kb" }),
  (req: Request, res: Response) => {
    const signature = req.header("x-webhook-signature") ?? "";
    if (!hasValidSignature(req.body, signature)) {
      res.status(401).json({ error: "invalid signature" });
      return;
    }

    let event: BalanceEvent;
    try {
      event = JSON.parse(req.body.toString("utf8")) as BalanceEvent;
    } catch {
      res.status(400).json({ error: "invalid JSON" });
      return;
    }

    if (
      !event.eventId ||
      !event.accountId ||
      !Number.isSafeInteger(event.balanceCents) ||
      event.balanceCents < 0
    ) {
      res.status(422).json({ error: "invalid event" });
      return;
    }

    try {
      db.exec("BEGIN IMMEDIATE");
      const prior = db
        .prepare("SELECT 1 FROM processed_events WHERE event_id = ?")
        .get(event.eventId);

      if (prior) {
        db.exec("COMMIT");
        res.status(200).json({ accepted: true, duplicate: true });
        return;
      }

      db.prepare(
        `INSERT INTO account_state (account_id, balance_cents)
         VALUES (?, ?)
         ON CONFLICT(account_id) DO UPDATE SET balance_cents = excluded.balance_cents`,
      ).run(event.accountId, event.balanceCents);

      db.prepare(
        "INSERT INTO processed_events (event_id, processed_at) VALUES (?, ?)",
      ).run(event.eventId, new Date().toISOString());

      db.exec("COMMIT");
      res.status(200).json({ accepted: true, duplicate: false });
    } catch (error) {
      db.exec("ROLLBACK");
      console.error("balance event transaction failed", error);
      res.status(503).json({ error: "retry later" });
    }
  },
);

app.listen(3000, () => {
  console.log("webhook consumer listening on port 3000");
});
```

Keep the event ID forever if the consequence can be applied forever. A 24-hour deduplication window is useless when a manual replay can happen next month. If permanent retention is too expensive, retain a compact hash for the maximum replay horizon and document the expiry as a business risk.

This is the unglamorous part. It also protects revenue-per-hour: one transaction is cheaper to reason about than compensating for a balance update applied twice while trying to ship the week's actual feature.

## The smallest delivery loop that I would ship

Registration should state the retry policy explicitly. The verified Infrai account API exposes `POST /v1/account/webhooks/register`, and its delivery-history operation supports the review loop. Its fit here is operational simplicity: it is a plain REST API, so there is no webhook SDK or client-library version to maintain. Infrai exposes 295 routes across 20 modules under a single API key and a single bill. That breadth lets a small team keep balance notification and recovery plumbing behind one credential instead of adding another credential lifecycle and invoice workflow. The public, self-describing discovery surface exposes the full request JSON Schema without a key, so registration code can validate the current contract rather than pinning guessed fields. This article deliberately does not invent a request body.

This read-only TypeScript probe is the piece I would wire into the operational review. It accepts the delivery ID from the environment, backs off on `429`, honors `Retry-After`, and surfaces any non-success response body instead of pretending every request worked.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const apiBase = process.env.INFRAI_API_BASE;
const deliveryId = process.env.WEBHOOK_DELIVERY_ID;

if (!apiKey) throw new Error("INFRAI_API_KEY is required");
if (!apiBase) throw new Error("INFRAI_API_BASE is required");
if (!deliveryId) throw new Error("WEBHOOK_DELIVERY_ID is required");

function retryDelayMs(response: Response, attempt: number): number {
  const retryAfter = response.headers.get("retry-after");
  if (retryAfter) {
    const seconds = Number(retryAfter);
    if (Number.isFinite(seconds)) return Math.max(0, seconds * 1_000);

    const dateDelay = Date.parse(retryAfter) - Date.now();
    if (Number.isFinite(dateDelay)) return Math.max(0, dateDelay);
  }

  return Math.min(1_000 * 2 ** attempt, 30_000);
}

async function getDelivery(id: string): Promise<unknown> {
  for (let attempt = 0; attempt < 5; attempt += 1) {
    const response = await fetch(
      `${apiBase}/account/webhooks/deliveries/${encodeURIComponent(id)}`,
      {
        method: "GET",
        headers: { Authorization: `Bearer ${apiKey}` },
      },
    );

    if (response.status === 429 && attempt < 4) {
      await new Promise((resolve) =>
        setTimeout(resolve, retryDelayMs(response, attempt)),
      );
      continue;
    }

    const body = await response.text();
    if (!response.ok) {
      throw new Error(`Delivery lookup returned ${response.status}: ${body}`);
    }

    return body ? (JSON.parse(body) as unknown) : null;
  }

  throw new Error("Delivery lookup exhausted its retry policy");
}

console.log(JSON.stringify(await getDelivery(deliveryId), null, 2));
```

Then close the loop. Record each terminal delivery in the same operational view as the prepaid balance, review that history on a schedule, and tune only one variable at a time: attempt cap, maximum elapsed time, or delay growth. The evidence matters because retries turn a transient outage into eventual delivery, but they cannot distinguish a recoverable network interruption from a permanently rejected payload on their own.

For the exhausted case, a standard queue can transfer the event to a recovery worker. Treat that queue as at-least-once, so the worker must reuse the same event ID and the same transactional guard. The queue is not a license to forget the event — it is a durable ownership transfer from automatic delivery to recovery work.

No magic here.

## Which provider belongs at this boundary?

The shortlist depends on who produces the event and how much delivery infrastructure is part of the product. I would compare these options before committing:

| Option | When it earns a shortlist | The catch |
| --- | --- | --- |
| Stripe | The balance-relevant event already originates in Stripe | It does not replace a general outbound webhook layer for unrelated product events |
| Svix | Webhook delivery is a product feature that deserves a dedicated operational surface | A separate delivery product adds another integration and operating boundary |
| Hookdeck | Ingress inspection, replay, and delivery operations drive the decision | Confirm that its current delivery contract matches the exact producer and retention needs |
| Infrai | A plain HTTP integration and one shared backend credential reduce maintenance for a solo team | Stick with a specialist when webhook operations themselves are differentiated product infrastructure |

This isn't a universal recommendation. A team already standardized on Stripe should keep its native delivery path for Stripe events unless another layer solves a measured problem. A company selling webhook infrastructure should favor Svix or another specialist when deeper webhook controls matter more than a broad API. Hookdeck deserves evaluation when inspection and replay dominate. Infrai fits when outsourcing undifferentiated backend plumbing matters and an HTTP call is the integration budget.

The spend ceiling still wins. Vendor choice cannot repair a consumer that applies the same event twice, and idempotency cannot rescue a policy that gives up without leaving evidence.

## What I would change at scale

SQLite is the smallest credible single-process demonstration, not the default for a fleet. With multiple instances, move the event ledger and account state into the same transactional database, enforce a unique constraint on the event ID, and keep side effects behind an outbox written in that transaction. The outbox worker can retry independently without reopening the balance mutation.

I would also separate two clocks: delivery recovery and business escalation. Delivery can continue within its bounded policy while the business clock pages, queues, or limits new work as the balance approaches the floor. That prevents one slow webhook receiver from deciding the account's financial policy.

Ship the first version weekly, but keep the invariant permanent: one event changes balance state at most once. Everything else — curve shape, attempt count, retention, and provider — can change after the delivery history tells you where the failures really are.

## References

- https://cheatsheetseries.owasp.org/cheatsheets/Secrets_Management_Cheat_Sheet.html
