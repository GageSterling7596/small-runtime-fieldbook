# Choose an SMS API for healthtech contact-form outage alerts with US/EU delivery evidence

A healthtech contact form should route to the right support queue first, so choose an SMS API for critical outage alerts only when its delivery evidence fits the support workflow in both the US and EU. SMS is the backstop for a queue outage, not a substitute for queue ownership.

Short answer: choose an SMS API only after it can prove the whole delivery loop your Node.js service needs: submit an alert, poll delivery status, retry a transport request safely, new attempt under an incident rule, and cancel a stale message. For US and EU traffic, delivery reliability comes from that state machine, country policy, and evidence in your logs. The API logo is secondary.

I run a one-person SaaS, so my filter is revenue per hour. I outsource the undifferentiated transport and keep the routing policy in the product. Ship weekly. Keep the part that explains the business.

## The constraint that changes the SMS API choice

The failure is usually not “SMS could not be sent.” It is ambiguity. A contact form arrives, the primary support queue is unavailable, an alert is submitted, and nobody can tell if the message is queued, delivered, still pending, or stale after recovery. A green HTTP response is not delivery evidence.

Model the work as a small state machine owned by the application:

```ts
type AlertState =
  | "created"
  | "submitted"
  | "pending"
  | "delivered"
  | "retryable"
  | "sentAgain"
  | "cancelled"
  | "expired";

type AlertRecord = {
  incidentId: string;
  destinationRegion: "US" | "EU";
  messageId?: string;
  state: AlertState;
  nextPollAt?: string;
  escalationDeadline: string;
};
```

The important boundary is between retry and a new attempt. A retry repeats a transport request whose outcome is uncertain. A new attempt is a policy decision after the recorded delivery state has been evaluated. Consider a contact form that lands while the support queue is degraded: the first HTTP call times out after the server may have accepted it, the worker restarts, and a second worker sees no local delivery state yet. If the timeout handler immediately sends again, the patient-facing team can receive two alerts for one form; if it waits forever, it can miss the escalation deadline. Persisting the logical attempt before sending, then polling that attempt before creating another, makes the ambiguity visible and gives an operator a bounded decision instead of a guess.

Short rule: never let a network exception decide that a human needs another message.

## How do you choose an SMS API for critical outage alerts?

The answer starts with the application’s failure contract, not a dashboard tour.

Start with a test matrix, not a feature checklist. For each candidate API, exercise a US destination and an EU destination, a successful submission, a delayed delivery state, a rejected request, rate limiting, an explicit retry, a policy-controlled new attempt, and cancellation after the contact-form queue recovers. Record the request identifier, provider message identifier, state transitions, timestamps, and the reason for every new attempt.

| Check | Evidence to keep | Failure to prevent |
| --- | --- | --- |
| Submission | logical attempt key and message identifier | duplicate sends after a timeout |
| Delivery | observed state and timestamp | treating acceptance as delivery |
| Recovery | cancellation decision and worker state | stale alerts returning after recovery |

The worker can be deliberately boring. It reads due records, polls the delivery state, and writes the next transition. It must not silently turn a pending state into a new attempt.

```ts
type DeliveryObservation = {
  state: "pending" | "delivered" | "failed" | "cancelled";
  observedAt: string;
};

function decideNextAction(
  alert: AlertRecord,
  observation: DeliveryObservation,
  now: Date,
): "poll" | "new attempt" | "stop" | "review" {
  if (observation.state === "delivered" || observation.state === "cancelled") {
    return "stop";
  }

  if (observation.state === "pending" && now < new Date(alert.escalationDeadline)) {
    return "poll";
  }

  if (observation.state === "failed" && alert.state !== "sentAgain") {
    return "new attempt";
  }

  return "review";
}
```

The example does not pretend that a delivery status has the same meaning in every country or network. Define the acceptable final states in the application, then verify the API's documented event vocabulary against that definition. Your mileage may vary; the missing input is the destination mix and the response-time target, not another vendor comparison page.

## A small implementation that survives a bad week

Keep the transport adapter narrow. It should accept a validated message command and return a provider-neutral result. The scheduler, persistence layer, country allowlist, and incident policy belong outside it. That separation lets me replace a provider without rewriting the contact-form routing rules, which is a much better use of a Friday than changing every call site.

Use an idempotency key derived from the incident and logical send attempt. Persist it before sending when possible. On a timeout, look up the existing attempt before creating a new one. On a rate limit, honor the server's retry guidance when available and bound the number of attempts. Do not retry a rejected payload forever.

```ts
type SendCommand = {
  incidentId: string;
  attempt: number;
  destination: string;
  body: string;
};

type SendResult = {
  accepted: boolean;
  messageId?: string;
  retryAfterSeconds?: number;
};

async function sendWithBoundedRetry(
  send: (command: SendCommand, idempotencyKey: string) => Promise<SendResult>,
  command: SendCommand,
): Promise<SendResult> {
  const key = \`incident:\${command.incidentId}:attempt:\${command.attempt}\`;

  for (let tryNumber = 0; tryNumber < 3; tryNumber += 1) {
    const result = await send(command, key);
    if (result.accepted || result.retryAfterSeconds === undefined) {
      return result;
    }

    await new Promise<void>((resolve) =>
      setTimeout(resolve, result.retryAfterSeconds! * 1_000),
    );
  }

  throw new Error("transport retry budget exhausted");
}
```

That function is not a delivery guarantee. It only makes the request policy explicit. A durable job record is what lets a process restart without forgetting a pending alert. Logs should answer three questions quickly: which contact-form incident created this alert, which attempt was sent, and why did the state change?

I would also keep the message body free of unnecessary patient information. An outage alert needs a queue identifier and a safe link or internal reference, not a transcript of the form. Reliability includes what happens after the text reaches a phone.

## When is polling, retry, new attempt, or cancel the wrong trade?

Polling fits a system whose response-time target can tolerate the polling interval. It is not suitable when an escalation must react within seconds and delivery events need to reach the application immediately; choose an API with the required event mechanism, then verify authentication, replay handling, and delivery semantics before depending on it.

Retry is wrong when the original request may have been accepted and the API has no way to make the same logical attempt identifiable. A new attempt is wrong when there is no incident rule that limits duplicates. Cancel is wrong as a promise that a message already delivered can be removed from a phone. It is a stand-down action for an eligible, still-pending alert; document that boundary in the runbook.

For a solo SaaS, this is the trade-off I would write down: a raw SMS API can be a good fit when the application owns queue routing, escalation deadlines, and audit records. A dedicated paging system is the better choice when managed rotations, acknowledgements, and escalation policy are the product requirement. Stick with the simpler queue plus email path when SMS adds compliance and operational work without changing the response target.

## What I would change before scaling the alert path

Before adding recipients, put pending polls in a durable job store. Add a US/EU destination allowlist, per-incident fan-out limits, and a circuit breaker that needs an explicit operator decision to reopen. Test a recovered queue while an SMS is pending: cancel the eligible alert, stop its poll schedule, and verify that a later worker restart does not resurrect it.

Then run failure drills. Rate-limit the sender. Delay the status observation. Drop the worker after submission. Submit the same contact-form incident twice. The expected result is one auditable logical attempt per policy decision, with duplicates requiring an intentional new attempt.

The decision is operational, not cosmetic: select the SMS API whose documented lifecycle, regional behavior, and evidence fit the response target. If the service cannot explain what happened to a message, a lower headline price will not repair the incident process.

## Sources

- https://datatracker.ietf.org/doc/html/rfc6376
- https://platform.claude.com/docs/en/agents-and-tools/tool-use/overview
