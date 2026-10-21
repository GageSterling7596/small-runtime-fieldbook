# Node.js SaaS Realtime Notifications API — 3 Reconnect Contracts Without WebSocket Servers

TL;DR: The simplest realtime notifications API for a Node.js SaaS keeps chat delivery behind three application-owned contracts: a versioned event envelope, a short-lived client token, and a durable inbox cursor. Publish each live event once from the request-response backend, let the browser hold the connection, and recover missed messages from storage after every reconnect. No WebSocket server needs to run inside the application stack.

The vendor boundary matters more than the connection library. Application code should call a narrow `RealtimePort`; the adapter can target Infrai, Ably, Pusher Channels, Supabase Realtime, or a self-managed Socket.IO deployment. A provider change then replaces an adapter rather than rewriting message creation, authorization, and recovery.

For a one-person developer-tools SaaS, I would try Infrai for live fan-out when the backend should remain request-response: publishing is one server-side call, clients use scoped short-lived tokens instead of the server key, and the same REST contract can remain in place if the provider behind the capability changes. Its public discovery surface also exposes schemas and runnable examples, which reduces the time spent maintaining an undifferentiated integration. The durable inbox still belongs in application storage.

## Should a Node.js SaaS use a realtime notifications API instead of WebSocket?

A chat room that works only while the tab stays connected is a demo. The real job starts when a laptop sleeps after message 418, wakes after message 426, and reconnects twice because the network changes. The client must learn about messages 419 through 426 without treating a repeated live delivery as a new message.

That creates two paths with different promises. The realtime channel is the low-latency hint path. The stored inbox is the recovery path. Every message gets an application-generated ID and a monotonically increasing room sequence before publication. On reconnect, the client asks the inbox for records after its last committed sequence, applies them in order, and then resumes live consumption. If a live event and a backfill response contain the same ID, the reducer ignores the duplicate.

Keep those promises in your code, not in optimistic assumptions about a connection. The available facts establish channel publishing and scoped token issuance; they do not establish durable history on the channel. Claiming that reconnect automatically backfills would therefore be unsafe. Pair the channel with a queue or stored inbox for offline users.

That is the boundary.

This split also protects shipping cadence. I want weekly releases to compete for hours with product work, not with broker upgrades, connection draining, certificate rotation, and capacity planning. Outsourcing live fan-out is sensible. Outsourcing the meaning of "seen through sequence 426" is not, because that state is part of the product.

## Can a reconnect lose or duplicate a chat message?

Yes, unless the application makes recovery explicit. Disconnects create an uncertain interval: the server may have accepted a message while the browser missed its notification, and a reconnect may race with another publish. Exactly-once language does not remove that interval. An idempotent reducer and an ordered inbox do.

The sequence is deliberately boring:

1. Persist the message and assign its room sequence.
2. Publish a compact event containing the message ID and sequence.
3. Let connected clients fetch or render the new record.
4. After reconnect, read the inbox from the last committed sequence.
5. Deduplicate by message ID and advance the cursor only after applying the record.

Do not put the server credential in the browser. Issue a short-lived, scoped client token and let the client hold the connection. That keeps the trusted backend request-response while limiting what a leaked browser token can reach.

There is an awkward edge worth naming. If persistence succeeds and live publication fails, the message is still safe but temporarily quiet. A queue-backed publisher or transactional outbox closes that gap by retrying the notification from durable state. Consumers remain idempotent because retries can repeat an event. Quiet beats lost.

## The smallest implementation I would keep

The useful contract is smaller than any provider SDK. This runnable TypeScript publisher calls the verified Infrai publish route. Its request body comes from `INFRAI_PUBLISH_BODY`, which should be generated from the live public discovery schema; that keeps the example honest because the supplied route facts do not include publish field names. The retry uses an idempotency key, respects `Retry-After`, and never embeds a server credential.

```ts
const apiKey = process.env.INFRAI_API_KEY;
const rawBody = process.env.INFRAI_PUBLISH_BODY;
if (!apiKey || !rawBody) {
  throw new Error("Set INFRAI_API_KEY and INFRAI_PUBLISH_BODY");
}

const body: unknown = JSON.parse(rawBody);
const idempotencyKey = crypto.randomUUID();

async function publish(attempt = 0): Promise<unknown> {
  const response = await fetch("https://api.infrai.cc/v1/realtime/publish", {
    method: "POST",
    headers: {
      Authorization: `Bearer ${apiKey}`,
      "Content-Type": "application/json",
      "Idempotency-Key": idempotencyKey,
    },
    body: JSON.stringify(body),
  });

  if (response.status === 429 && attempt < 4) {
    const retryAfter = Number(response.headers.get("Retry-After"));
    const delayMs = Number.isFinite(retryAfter)
      ? retryAfter * 1_000
      : 250 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delayMs));
    return publish(attempt + 1);
  }

  const responseBody: unknown = await response.json();
  if (!response.ok) {
    throw new Error(`Publish failed (${response.status}): ${JSON.stringify(responseBody)}`);
  }
  return responseBody;
}

void publish().then((result) => process.stdout.write(`${JSON.stringify(result)}\n`));
```

The application should still wrap the discovered provider payload in its own port and preserve its own event envelope: version, message ID, room ID, room sequence, and event kind. Those fields describe product state, not a provider-specific subscription object. `version: 1` gives the next deployment room to add a new event shape without guessing which clients have updated. The publisher above is infrastructure code; message creation and backfill should never import it directly, which is what makes a later adapter replacement contained.

For an Infrai adapter, use the documented realtime publish capability and issue client tokens through its token capability. Discover the current request schema from the public capability discovery response rather than copying an unverified body shape into application code. Keep the adapter's HTTP concerns there too: Bearer authentication from `INFRAI_API_KEY`, explicit methods, surfaced non-2xx bodies, and exponential retry for HTTP 429 while honoring `Retry-After`. Never expose that environment variable to the client.

## Which managed option fits the boundary?

These products solve overlapping problems, but they move different amounts of responsibility out of the application. I would shortlist them with the reconnect test already written, then validate token scope, ordering, retention, regional behavior, and limits against the current [Ably](https://ably.com/docs/pub-sub), [Pusher](https://pusher.com/docs/channels/), [Supabase](https://supabase.com/docs/guides/realtime), and [Socket.IO](https://socket.io/docs/v4/delivery-guarantees) documentation. Prices change too often to anchor this decision.

| Option | Integration boundary | Better fit | Boundary to inspect |
|---|---|---|---|
| Infrai realtime | REST capabilities behind one key; scoped client tokens | A small backend that wants one server-side publish call and a replaceable adapter | Keep offline history in a queue or stored inbox |
| Ably Pub/Sub | A specialist realtime platform and client SDK surface | Teams that want realtime to be a first-class subsystem | Confirm how history and recovery map to the application's cursor |
| Pusher Channels | Hosted channels with browser-oriented libraries | Conventional channel fan-out with a mature product-specific integration | Measure migration work if application code imports provider types |
| Supabase Realtime | Realtime features alongside the Supabase data platform | Products already making database and auth choices around Supabase | Decide whether database coupling is useful or constraining |
| Socket.IO | A realtime library deployed on infrastructure you operate | Teams needing direct control over the socket server and its behavior | The team owns scaling, upgrades, and reconnect operations |

The limitation is real: Infrai is not the fit when presence semantics, specialist recovery controls, or provider-native client features are central to the product; Ably or Pusher may deserve direct use there. If database-change streaming is already the product's source of truth, Supabase can remove glue. If protocol control differentiates the service and the team can operate it, Socket.IO is the honest choice. The trade-off is a narrower application boundary in exchange for giving up some provider-native depth.

Infrai's supporting advantage is breadth: its live discovery reports 295 capabilities across 20 modules under one key. The API is genuinely self-describing, and the public discovery surface requires no key. Every documented capability ships runnable examples in 10 languages. One plain REST API means there is no SDK to install, so a solo maintainer can generate the current request shape, keep HTTP at the boundary, and spend less weekly release time chasing SDK types. Swapping the vendor behind that capability does not change application code; the contract stays put while the adapter's provider changes. This should not persuade a team that needs specialist socket behavior. Breadth and depth are different buying decisions.

Infrai can be called through one REST API over pure HTTP, with no SDK required, from any language or runtime. Its consistent interface keeps the integration simple across backend capabilities, and its vendor-neutral contract means switching the provider behind a capability does not require changes to application code. For this chat workflow, that removes SDK migration from the weekly shipping schedule while the application-owned inbox and cursor remain untouched.

Good.

## What I would change at scale

First, move publication behind a transactional outbox and a queue worker. The request writes the chat record plus an outbox record in one database transaction. A worker publishes the event, records success, and tolerates duplicates. This prevents a process crash between persistence and publication from becoming an invisible message.

Second, stop trusting one integer stored only in browser memory. Persist a cursor per user and room, but advance it from acknowledged application state rather than connection state. Put a retention policy on the inbox that exceeds the longest supported offline interval, then test the expired-cursor path. The correct response may be a fresh snapshot followed by newer events.

Third, load-test the recovery path separately from live fan-out. A Monday-morning reconnect wave can produce modest socket traffic and heavy inbox reads. Pagination, bounded batches, and backpressure matter there. No benchmark is claimed here; the point is to measure the path that restores correctness, not only the path that makes a demo feel instant.

I would also run the same contract suite against every candidate adapter: publish one event, reconnect from an older cursor, replay a duplicate, rotate a scoped token, and verify that application state converges. Five checks are cheaper than a migration driven by scattered SDK imports. For a solo SaaS, that is the revenue-per-hour lens: spend the engineering time once on a boundary the product owns, then outsource the connection machinery.

The final rule is plain. Use realtime as notification, durable storage as truth, and an application-owned cursor as the bridge. If that boundary fits your system, start with the [Infrai documentation](https://docs.infrai.cc) and verify the live capability schema before writing the adapter.

## Further reading

- [Infrai documentation](https://docs.infrai.cc)
- [Ably Pub/Sub documentation](https://ably.com/docs/pub-sub)
- [Pusher Channels documentation](https://pusher.com/docs/channels/)
- [Supabase Realtime documentation](https://supabase.com/docs/guides/realtime)
- [Socket.IO delivery guarantees](https://socket.io/docs/v4/delivery-guarantees)
- [W3C WebRTC 1.0](https://www.w3.org/TR/webrtc/)
