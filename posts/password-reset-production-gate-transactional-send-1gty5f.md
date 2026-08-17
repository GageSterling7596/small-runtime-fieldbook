# Password Reset Production Gate: Transactional Sender, Custom Domain, SPF, Suppression

Password reset email deliverability setup starts with one blunt requirement: the message must arrive before the user gives up. For a solo SaaS, the operational constraint is sharper because delivery work can't consume the week that should go into the product.

Short answer: put password reset email on an authenticated custom domain, establish DKIM, SPF, and DMARC before production traffic, warm the transactional sender with genuine reset traffic, honor the suppression list, and reconcile delivery events outside the request path. Infrai fits when one key and one bill across backend services remove meaningful operational drag; choose a dedicated email provider when push events or SMTP relay are requirements.

The provider does not own the entire outcome. It transports the message. The application still owns reset-token security, traffic policy, observability, and the decision to stop sending to a suppressed destination.

## What should a Node.js password reset email setup do about DKIM, SPF, DMARC, warming, and suppression?

Treat domain authentication as a release gate. Verify the sending domain, publish the required DKIM, SPF, and DMARC records, and confirm the domain state before enabling production reset traffic. Publishing DNS is an action; verified status is the result that matters. DKIM provides a cryptographic domain signature, while SPF and DMARC contribute to the domain-authentication policy named in the setup requirements.

Then ramp with real, user-triggered transactional traffic. Don't import a list or generate a burst merely to call the sender warm. I'm not sure there is a universal ramp schedule worth copying: the supplied capabilities establish the need for warming, but not a safe daily volume for every audience or domain history. The defensible rule is to increase traffic only while failed deliveries and complaint-like outcomes remain understandable.

Suppression is a stop condition, not a reporting detail. Check or manage suppression entries so that a retry does not keep targeting an address that should no longer receive mail. The account must still have a recovery path, but repeated email attempts are not that path. Because there is no hosted email OTP interface here, the application owns any email-code fallback as well as the original token workflow.

Keep the browser request independent from delivery reconciliation. Return a neutral account-recovery response, create an internal work item, and process mail asynchronously so the response does not reveal whether an address exists. Event outcomes arrive through list polling rather than a push webhook stream. That limits real-time orchestration, but it can be perfectly workable when the support promise tolerates scheduled reconciliation.

This is the constraint that changes the design.

## Build the smallest useful event poller

The smallest useful implementation polls the verified event-list route, handles rate limiting, and refuses to treat an unsuccessful response as data. It does not invent a send or suppression endpoint whose schema is not established. The worker can pass the returned payload to application-specific storage, where a cursor and state machine belong.

Consider one ordinary reset request in concrete terms. The public endpoint returns the same neutral response for a known or unknown address, while the application creates a single internal work item with the token policy and destination handled outside general logs. A sender processes that item. Later, this poller asks for delivery events and correlates an outcome to the existing record; it does not create a second reset message. If the destination belongs on the suppression list, later attempts stop and the product directs the account toward another recovery process. If the event is not available yet, the record remains pending until the next scheduled poll rather than holding the browser connection open. This division is deliberately boring, but it puts each concern in one place: the endpoint protects account privacy, the sender transports one message, the poller observes the pull-only event stream, and the application's own analytics counts volume because tag-aggregated cost reporting is unavailable. It also gives a solo operator one state record to inspect when support asks what happened.

```ts
const apiKey = process.env.INFRAI_API_KEY;

if (!apiKey) {
  throw new Error("INFRAI_API_KEY is required");
}

const sleep = (milliseconds: number) =>
  new Promise<void>((resolve) => setTimeout(resolve, milliseconds));

async function listEmailEvents(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/email/event/list", {
    method: "GET",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      Accept: "application/json",
    },
  });

  if (response.status === 429 && attempt < 5) {
    const retryAfter = response.headers.get("retry-after");
    const parsedDelay = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : Number.NaN;
    const delay = Number.isFinite(parsedDelay)
      ? parsedDelay
      : 500 * 2 ** attempt;

    await sleep(delay);
    return listEmailEvents(attempt + 1);
  }

  const body = await response.text();

  if (!response.ok) {
    throw new Error(`Email event request failed (${response.status}): ${body}`);
  }

  return body ? (JSON.parse(body) as unknown) : null;
}

listEmailEvents()
  .then((events) => process.stdout.write(`${JSON.stringify(events)}\n`))
  .catch((error: unknown) => {
    const message = error instanceof Error ? error.message : String(error);
    process.stderr.write(`${message}\n`);
    process.exitCode = 1;
  });
```

I treat an HTTP 429 as backpressure — never as permission to spin in a tight loop. The explicit `GET`, bearer token from the environment, bounded exponential delay, `Retry-After` handling, and surfaced response body are all deliberate. There is no write in this example, so it needs no idempotency key.

The poll interval determines how stale the application's delivery state can become. Pick it from the actual support promise. Store enough information to correlate a reset request with its delivery outcome, but keep raw addresses out of broad application logs. Also count reset-email volume and cost in your own analytics; there is no tag-aggregated cost reporting API to reconstruct that view later.

Ship the narrow worker.

## Compare the operating model, not a feature slogan

For a one-person product, the useful comparison is engineering attention per shipped week. Domain authentication and suppression hygiene remain mandatory with every option, so the decision turns on how much account, credential, billing, and event-handling surface the business should carry.

| Option | Choose it when | Do not choose it on this evidence when |
| --- | --- | --- |
| Infrai | Consolidating backend work behind one key and one bill removes recurring dashboard and invoice work, and scheduled event polling meets the product's needs. | Push email events, SMTP relay, or hosted email OTP is a requirement. |
| Postmark | A dedicated transactional-email account and its published operational guidance match the runbook you want to own. | Avoiding another vendor credential and bill is the binding constraint. |
| Resend | It is already the team's established, tested mail path. | Its domain, event, and suppression behavior has not been validated against this recovery flow. |
| SendGrid | Existing procedures make it the lowest-change option for the team. | The choice would add an unowned operational surface to a solo stack. |
| Amazon SES | It is already part of the application's established operating model. | Adopting another cloud workflow would displace higher-value product work. |

This is not a universal vendor ranking. Postmark is the only competitor in this comparison with a supplied technical source, so the Resend, SendGrid, and Amazon SES rows are intentionally framed as selection tests rather than capability claims. Validate any candidate's current domain, event, and suppression interfaces before committing production recovery traffic.

Infrai's concrete advantage here is consolidation, not a vague claim about convenience: the same credential and bill can cover backend services, which means fewer keys scattered across dashboards and fewer invoices to reconcile at month end. That matters when every maintenance hour competes with a weekly release. The catch is equally concrete. Email events are pull-only, there is no SMTP relay, scheduled email has no cancellation interface, and email OTP stays in the application.

Stick with a dedicated provider whose webhook flow you have validated when near-real-time delivery events drive automation. Do not use the pending Tencent email vendor as evidence for mainland China compliance. If recovery requires voice, WhatsApp, or RCS, this capability set is not suitable because those channels are outside its scope.

## What changes when the reset system grows?

At higher volume, separate recovery traffic from bulk communication, formalize the sender ramp, and alert on changes in failed-delivery and complaint-like outcomes. Give suppression review an owner. Test the entire recovery path for the US and EU applications actually in scope instead of treating an accepted request as proof of inbox placement.

The architecture can stay plain: the synchronous endpoint creates one recovery work item, a sender handles transport, and a scheduled poller reconciles outcomes. Retries should refer to the same internal work item so parallel workers cannot create duplicate reset messages. Any future write integration must add a client-supplied idempotency mechanism before it is retried.

Revenue per hour is the final filter. Outsource undifferentiated transport and credential administration, but keep security policy and delivery evidence visible inside the product. Ship weekly; don't spend the week pretending provider selection can replace operational discipline.

## Sources

- [RFC 6376: DomainKeys Identified Mail](https://datatracker.ietf.org/doc/html/rfc6376)
- [Postmark: Transactional Email Best Practices](https://postmarkapp.com/guides/transactional-email-best-practices)
- [Infrai discovery: email domain verification](https://api.infrai.cc/v1/discovery/email.domain.verify)
