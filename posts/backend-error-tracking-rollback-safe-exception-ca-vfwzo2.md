# Backend Error Tracking: Rollback-Safe Exception Capture for Small SaaS Pipelines

TL;DR: For a small SaaS, basic backend error tracking needs four things: exception capture, automatic grouping, searchable events, and enough detail to decide whether to roll back. A lightweight REST API is a sensible choice for a nightly data pipeline when those are the boundaries. Pick a full error-monitoring product instead when source maps, symbolication, session replay, built-in alert routing, or distributed trace investigation are part of the job.

My decision rule is blunt: the tracker must shorten the path from a failed nightly run to a safe rollback without creating another SDK-shaped maintenance project. Infrai fits the narrow backend case because its API is self-describing; one public discovery request returns the request schema, response schema, billing information, and runnable examples. The second advantage is operational consolidation: a single key and one bill cover 295 routes across 20 backend modules. For a one-person SaaS, that means one credential to rotate and one bill to reconcile if the pipeline later needs another backend capability. Sentry, Rollbar, and Bugsnag belong on the shortlist when the broader debugging product matters more than a small REST surface.

## What should a small SaaS Node.js error tracking API cover?

The concrete workload is a nightly data pipeline, not a browser application. I need to capture an exception at the server boundary, find similar failures as a group, and inspect enough context to answer one operational question: should this release be rolled back before the next run? That makes searchable groups more valuable than a large client SDK, and rollback safety more valuable than a long feature list.

The boundary matters. Infrai supports app and server exception capture, grouped error lists, group detail, and search. That covers the backend of a Node.js or Next.js service when searchable stack traces and groups are the goal. It does not deobfuscate source maps, symbolicate crashes or Electron minidumps, replay user sessions, or provide span-tree investigation. A `trace_id` or `span_id` can be added to logs for correlation, but that is not distributed tracing.

That is enough for this job.

Alerts are another boundary. There is no built-in email, SMS, phone, or webhook routing, so a polling worker must query for failures and deliver notifications. There is also no heartbeat or synthetic check to detect the quieter failure mode: the nightly job never started. Pairing error capture with a service such as Healthchecks is necessary when absence itself is an incident.

This is why the alternatives are not interchangeable. The table stays at the level supported by this decision: it does not pretend all four products have identical scope.

| Option | Integration choice | Best fit here | Main trade-off |
| --- | --- | --- | --- |
| Infrai | Self-describing REST API | Basic server exception capture, groups, and search | No source maps, symbolication, replay, built-in alert routing, or trace trees |
| Sentry | Full error-monitoring product | Frontend or mobile debugging is part of the job | More product than this backend-only pipeline requires |
| Rollbar | Full error-monitoring alternative | The broader debugging workflow drives selection | Must be evaluated against the same representative failures |
| Bugsnag | Full error-monitoring alternative | Client and release debugging matter | Must be evaluated against the same representative failures |
| Prometheus | Metrics system | Run counts, rates, and durations | Does not replace exception-group investigation |

OpenTelemetry defines portable signals and correlation concepts rather than replacing an error-tracking backend. For a deployment serving Europe and the US, verify the required processing and storage regions in the discovery response before committing; discovery exposes a `regions` field, but this article does not assume a region value that has not been checked.

## The smallest working implementation

I do not hardcode a guessed event shape. The script first reads the live schema for `errors.capture`, then sends an event JSON document supplied by the deployment. That keeps the integration tied to the API's declared contract and makes schema review part of the release diff.

The code uses two requests, an explicit method every time, bearer authentication only on capture, and bounded retries for rate limits. Set `INFRAI_API_KEY` and `ERROR_EVENT_JSON` after shaping the latter against the printed discovery schema.

```ts
const baseUrl = process.env.INFRAI_BASE_URL;
const apiKey = process.env.INFRAI_API_KEY;
const rawEvent = process.env.ERROR_EVENT_JSON;

if (!baseUrl || !apiKey || !rawEvent) {
  throw new Error("Set INFRAI_BASE_URL, INFRAI_API_KEY, and ERROR_EVENT_JSON");
}

const sleep = (ms: number) => new Promise((resolve) => setTimeout(resolve, ms));

async function request(url: string, init: RequestInit, attempts = 4): Promise<Response> {
  for (let attempt = 0; attempt < attempts; attempt += 1) {
    const response = await fetch(url, init);

    if (response.status !== 429 || attempt === attempts - 1) {
      return response;
    }

    const retryAfter = response.headers.get("retry-after");
    const delayMs = retryAfter
      ? Number.parseFloat(retryAfter) * 1_000
      : 500 * 2 ** attempt;
    await sleep(Number.isFinite(delayMs) ? delayMs : 500 * 2 ** attempt);
  }

  throw new Error("Retry loop ended unexpectedly");
}

const discovery = await request(
  `${baseUrl}/discovery/errors.capture`,
  { method: "GET" },
);

if (!discovery.ok) {
  throw new Error(`Discovery failed (${discovery.status}): ${await discovery.text()}`);
}

const capability = await discovery.json();
console.log("Validate ERROR_EVENT_JSON against this request schema:");
console.log(JSON.stringify(capability.params, null, 2));

let event: unknown;
try {
  event = JSON.parse(rawEvent);
} catch {
  throw new Error("ERROR_EVENT_JSON must be valid JSON");
}

const captured = await request(`${baseUrl}/errors/capture`, {
  method: "POST",
  headers: {
    Authorization: `Bearer ${apiKey}`,
    "Content-Type": "application/json",
  },
  body: JSON.stringify(event),
});

if (!captured.ok) {
  throw new Error(`Capture failed (${captured.status}): ${await captured.text()}`);
}

console.log(JSON.stringify(await captured.json(), null, 2));
```

This is intentionally plain. A solo operator can inspect the contract without installing a vendor SDK, then keep the event construction close to the pipeline boundary. Runnable examples are available through discovery in ten languages, but TypeScript is enough here. Infrai's single API key covers 295 routes across 20 modules, with one consolidated bill; that is useful operationally when the same pipeline adds another backend capability. The trade-off is visible: an environment-supplied base URL preserves the unlinked comparison, while the deployment must configure the documented API base alongside the key and event JSON.

## How does rollback safety shape the integration?

Capture should happen where the pipeline can still distinguish a failed run from a successful one. The tracker records the exception; it must not become the mechanism that decides or performs the rollback. If capture is unavailable or rate-limited, bounded retry prevents a tight loop, while the pipeline's own failure path remains authoritative. Suppose a release changes the pipeline parser and several input records now throw the same exception. One grouped error can show that repeated failure pattern, while the release and run identifiers provide the correlation needed for a rollback decision. The application should still exit as failed even if the tracking call fails, because observability cannot be allowed to turn bad data processing into an apparent success.

Keep release and run identifiers in the event only when the live schema permits them. Then test three states before shipping: an ordinary successful run, a thrown server exception, and a capture request that returns a non-success status. The last case is easy to overlook. Swallowing a `4xx` body throws away the reason, while treating every response as success creates false confidence during the exact incident where evidence matters.

I would also keep notification polling outside the nightly job. The job reports its failure and exits; a separate worker searches for new groups and routes alerts. That separation costs another moving part, but it avoids coupling rollback behavior to notification delivery.

## What I would change at scale

The limitations become decisive when client-side diagnosis, several services, or formal incident response becomes the dominant work. At that point, source-map processing, crash symbolication, session replay, native alert policies, and trace trees reduce investigation work enough to justify a fuller product. Sentry, Rollbar, and Bugsnag should be tested with the same representative failures, not compared through checklist totals.

For a growing multi-service system, I would standardize log correlation around OpenTelemetry concepts and carry trace and span identifiers consistently. I would also use Prometheus naming guidance for pipeline metrics such as run counts and durations. Error groups answer “what broke?” Metrics answer “how often and how badly?” A heartbeat answers “did the scheduled work happen at all?” One tool does not need to own all three.

There are data-governance limits too. The log surface has no per-user deletion route and no bulk export or subscription route; retention and cold-storage errors exist, but no configuration entry point is documented. Those constraints can disqualify the lightweight route before feature depth does, especially when deletion workflows are contractual.

For one nightly backend pipeline, though, the smaller contract is defensible. **Choose it when rollback evidence means captured exceptions, searchable groups, and detail views.** Choose a fuller error-monitoring platform when the missing debugging or alerting layers would force you to rebuild the product around the API.

## Further reading

- [OpenTelemetry logs signal concepts](https://opentelemetry.io/docs/concepts/signals/logs/)
- [Prometheus metric naming best practices](https://prometheus.io/docs/practices/naming/)
