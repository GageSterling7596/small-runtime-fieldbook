# Postgres pg_cron vs App Cron Webhooks for Scheduled Data Cleanup

**Short answer:** For scheduled data cleanup in a normal app, use app cron to call an authenticated webhook; keep `pg_cron` for a small, database-only cleanup, and add a queue when one run can exceed a safe batch.

| Choice | Delivery boundary | Best fit | Main catch |
| --- | --- | --- | --- |
| Postgres `pg_cron` | The database starts SQL | One database, one bounded SQL operation | Application policy and deployment can split across two control planes |
| App cron plus webhook | A timer delivers an HTTP request | Cleanup needs tenant rules, an audit record, or application tests | The endpoint needs authentication, replay defense, and reachable networking |
| Scheduler plus queue workers | A timer starts durable units of work | A renewal campaign or purge spans many properties | More moving parts, plus duplicate delivery must be harmless |

The recommendation is app cron plus a narrow webhook for the common case. It keeps the business deadline, the cleanup policy, and the deployment in application code. For a one-person SaaS, that usually protects revenue per engineering hour better than maintaining database-side logic that the normal release path doesn't exercise.

Cheapest is the wrong first sort. The cheapest system is the one whose missed or repeated delivery you can detect and repair without losing a week of feature work.

## How reliable is scheduled data cleanup with Postgres pg_cron or an app cron webhook?

Choose by delivery guarantee and ownership, not by the number of setup steps. A property-management renewal reminder makes the distinction concrete. Suppose a lease should receive a reminder at a business deadline, while expired delivery receipts should be removed later under a separate retention rule. Both jobs involve time, but they have different consequences.

The reminder is business behavior. A timer may fire twice, a request may be retried, and a worker may restart after sending the message but before recording completion. The application must therefore own a stable operation ID, such as `renewal:lease_482:2026-09-30`, and enforce uniqueness where it records the send. Picture the awkward sequence: the scheduler makes request A, the handler creates the intent, a worker sends the reminder, and the process exits before its final status update; request B then arrives with the same operation ID. A correct receiver reads the existing intent and reconciles its state instead of creating a second send. At-least-once delivery can produce an effectively once-only business outcome because identity survives every hop. Cleanup needs the same discipline for a different reason: success destroys evidence, so record the run ID, fixed cutoff, scope, status, and counts before deleting anything, then select rows in bounded batches. A retry must reuse the original cutoff rather than calculate a fresh `now()`, or the repeated run can silently widen its target set. This is where app-level code earns its extra HTTP hop: policy checks, idempotency, logs, and deployment all live in the same reviewable path. Don't confuse the scheduler's successful dispatch with completion of either job.

Keep it boring.

`pg_cron` is a sensible runner-up when the whole operation is one bounded SQL task, the database team already owns scheduled maintenance, and no application rule or external effect is involved. It removes the HTTP boundary. The trade-off is that tests, rollout, permissions, and observability now straddle the application and database. That cost is small in a database-led operation and surprisingly expensive in a tiny team that ships weekly.

## Retention governance starts before the clock fires

No clock can promise exactly-once effects across a network and a database by itself. Design for an attempted delivery to be missing, late, or repeated. The receiver supplies the guarantee that matters to the business.

That's the test.

For a renewal deadline, write an immutable intent row before work begins. Give it a unique key derived from the lease and deadline. A worker claims the row, performs the bounded action, and records the result. If the same trigger arrives again, the unique constraint turns it into a read of the existing intent rather than a second reminder. If a run never reaches the receiver, an overdue-intent query exposes the gap. That is a repairable state; an unobserved calendar tick is not.

The same model works for deletion, but a cleanup run should also freeze its cutoff and selection rule. Process a limited number of rows per transaction so maintenance doesn't become an unbounded lock or load event. Store detailed evidence in the application database or logging system, not only in a scheduler's run page. I don't know your acceptable batch size; table shape, indexes, row width, and production load decide it. A staging run against representative data, followed by a deliberately small production canary, resolves that uncertainty.

If the job grows into one unit per property or table, put those units on a queue. Google Cloud Pub/Sub documents asynchronous, decoupled publisher and subscriber delivery as a messaging model; that is useful evidence for the architecture, not a reason to select one provider. The consumer still needs idempotency because messaging does not define your business outcome. Start with the webhook, measure the work, and introduce the queue only when bounded synchronous execution stops fitting the operational window.

There is a security edge too. A public trigger endpoint should accept one narrow command, not arbitrary SQL or caller-selected cutoffs. Sign a canonical payload with a shared secret using HMAC, include a timestamp and stable operation ID, compare signatures without timing leaks, and reject stale requests. RFC 2104 specifies HMAC's keyed-hashing construction. Replay policy, key rotation, and which fields are canonical remain application decisions.

## Implement the narrow TypeScript receiver

This example shows the control boundary rather than a vendor API. It verifies an HMAC over the raw body, checks a five-minute timestamp window, accepts a server-defined cleanup name, and passes a stable operation ID to a durable store. The store must enforce uniqueness on `operationId`; the in-process function should never be the only replay defense.

```ts
import { createHmac, timingSafeEqual } from "node:crypto";
import type { Request, Response } from "express";

type CleanupRequest = {
  operationId: string;
  cleanup: "expired-renewal-receipts";
  requestedAt: string;
};

type CleanupStore = {
  createOrGet(input: CleanupRequest): Promise<{ id: string; created: boolean }>;
};

function validSignature(rawBody: Buffer, suppliedHex: string, secret: string): boolean {
  const expected = createHmac("sha256", secret).update(rawBody).digest();
  const supplied = Buffer.from(suppliedHex, "hex");
  return supplied.length === expected.length && timingSafeEqual(supplied, expected);
}

export function scheduledCleanup(store: CleanupStore, secret: string) {
  return async (req: Request, res: Response): Promise<void> => {
    const rawBody = req.body as Buffer;
    const signature = String(req.header("x-cleanup-signature") ?? "");
    if (!validSignature(rawBody, signature, secret)) {
      res.status(401).json({ error: "invalid signature" });
      return;
    }

    const input = JSON.parse(rawBody.toString("utf8")) as CleanupRequest;
    const requestedAt = Date.parse(input.requestedAt);
    const ageMs = Math.abs(Date.now() - requestedAt);
    if (!Number.isFinite(requestedAt) || ageMs > 5 * 60 * 1000) {
      res.status(400).json({ error: "stale request" });
      return;
    }

    const plan = await store.createOrGet(input);
    res.status(plan.created ? 202 : 200).json({
      cleanupRunId: plan.id,
      accepted: true,
      duplicate: !plan.created,
    });
  };
}
```

The handler deliberately does not accept a cutoff or a free-form table name. The durable worker derives the permitted cutoff from the named retention policy, saves that value with the run, and deletes one batch at a time. Test a valid signature, an altered body, an expired timestamp, and two requests with the same operation ID. Then deploy the handler before enabling the schedule. Roll back by pausing new triggers while preserving already-created intents for inspection.

## Cost decides the runner-up only after recoverability

Stick with database scheduling when all of these are true: the work is local to one database, one SQL transaction can stay bounded, database migrations own the schedule, and database-native monitoring is already part of operations. A periodic purge of disposable staging rows can fit. An app webhook adds a network and authentication boundary without adding much clarity in that case.

It is not suitable when the deadline depends on property time zones, lease state, tenant-specific policy, or an external message. Those rules belong beside the application model and tests. Likewise, move from a direct webhook to queued workers when one invocation must fan out across many properties or when the work duration has become unpredictable. The queue is the runner-up for workload shape, not a default badge of seriousness.

Cost comes last. Compare the incremental timer and message charges with the hours required to operate another runtime, database extension, or queue. I'm not sure a hosted timer is cheaper for your traffic, because the supplied evidence contains no comparable pricing and usage patterns vary. The decision stays simple: use the control plane you already operate until delivery evidence or workload size proves it inadequate.

Ship the smallest design that leaves a durable intent, makes duplicates harmless, and exposes missed deadlines. For most application-owned cleanup, that is an authenticated webhook. For contained SQL maintenance, it is database cron. The boundary is ownership and recoverability, not fashion.

## Further reading

- https://www.rfc-editor.org/rfc/rfc2104
- https://cloud.google.com/pubsub/docs/overview
