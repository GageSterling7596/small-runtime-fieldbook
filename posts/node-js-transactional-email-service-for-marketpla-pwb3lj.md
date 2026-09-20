# Node.js Transactional Email Service for Marketplace Onboarding (Templates in Git)

A startup choosing a Node.js transactional email service for marketplace onboarding and contact routing should keep its templates in the application repository and put the delivery API behind a small interface. That choice costs a little engineering time now, but it keeps queue rules, message copy, and deployment history in one reviewable place. For a one-person SaaS, that is the useful trade: outsource delivery infrastructure, retain the business logic that changes when the marketplace changes.

| Choice | Template owner | Best fit | Main cost |
|---|---|---|---|
| Repository-rendered | Application | Routing rules and copy ship together | The application must render and validate content |
| Provider-rendered | Delivery service | Non-developers change copy often | Releases and template versions can drift |
| Split ownership | Both | Stable layout with a few remote content blocks | Two sources of truth must be tested |

**Recommendation:** render in the application, send through an API rather than SMTP, and make the delivery adapter replaceable. Do not choose on the smallest advertised unit price. Choose the ownership model whose failures you can diagnose during a support shift.

TL;DR: the cheapest easy option is the one that leaves the founder with one template history, one routing decision, and one retry boundary. For this marketplace contact-form job, that means repository-owned templates and a narrow HTTP transport contract.

## Should a startup own its transactional onboarding email templates?

A contact form looks simple until one submission has to reach the right support queue. Category, seller region, order state, and language may all affect routing. The email subject and body often explain why the ticket landed there. If routing lives in Node.js while the corresponding wording lives in a remote dashboard, one behavior has two release histories.

That split consumes founder time. A copy edit can describe a queue that the deployed code does not yet use, or a routing change can land without its matching explanation. Repository ownership makes the template, tests, and decision table part of the same review. A weekly release can move them as one unit.

Keep the rule small. For example, a marketplace might accept three explicit categories: `order`, `seller`, and `safety`. Unknown input should go to a reviewed fallback queue rather than becoming an arbitrary email address. The browser never selects the final recipient; the server does.

There is another boundary here: the `From` domain. SPF defines how a receiving system can check whether an SMTP client is authorized to use a domain in the `MAIL FROM` identity. RFC 7208 also limits the terms that cause DNS queries during an SPF check to 10. That is a concrete reason to treat domain authentication as infrastructure configuration, not something each template or request may improvise.

Short rules win.

Repository ownership also makes escaping visible. Contact-form text is untrusted input, even when it is headed for an internal queue. Render it as text or escape it before inserting it into HTML. Never let a submitted category become a header, recipient, or template identifier without an allowlist.

## The second criterion is operational ownership

An API call returning success is not the same event as an email reaching a queue. The application needs its own durable delivery state: a stable message key, the selected queue, the template revision, an attempt count, and the transport's opaque receipt. That record answers the expensive question later: did routing fail, did rendering fail, or did the delivery system reject the request?

The write path should be short. Validate the form, calculate the queue, persist the contact request and an outbox item in one local transaction, then return to the user. A worker renders and sends afterward. This prevents a slow external request from holding the form open, and it gives retries a durable starting point. Retry only ambiguous or temporary transport failures. A malformed address or rejected payload needs inspection, not five identical attempts; use the stable message key as the idempotency key when the selected API supports that concept, otherwise deduplicate in the adapter and store the remote receipt. Never generate a fresh logical message merely because a process restarted. The trade-off is extra local state, plus a worker that needs monitoring, in exchange for keeping an external timeout away from the user's form submission.

Observability can stay plain. Log the internal message key, queue code, template revision, attempt number, outcome class, and receipt. Do not log the contact message, magic link, or full recipient address. If an onboarding message contains an authentication secret, the bar is higher: NIST SP 800-63B says verifiers must accept a given one-time password only once while it is valid, and a single-factor OTP must contain at least six decimal digits. Delivery logs are the wrong home for either the password or its link equivalent.

That distinction matters. A welcome note can usually be retried as the same logical message. An authentication email must also respect the authenticator's lifetime and replay rules. Email delivery and identity verification are adjacent systems, not one feature. The 10-DNS-query SPF processing ceiling and the six-digit minimum for a single-factor OTP are hard constraints, while `contact-v3` is an application choice that can change in the next weekly release.

Do not blur them.

## A narrow Node.js boundary

The application does not need to know a delivery service's payload everywhere. One adapter is enough. The business layer supplies an already-rendered message and a stable key; the adapter translates it to the chosen HTTP API.

```ts
type Queue = "orders-eu" | "orders-us" | "seller" | "safety" | "triage";

type ContactRequest = {
  id: string;
  category: "order" | "seller" | "safety" | "unknown";
  market: "EU" | "US";
  replyTo: string;
  message: string;
};

type OutboundEmail = {
  messageKey: string;
  to: string;
  replyTo: string;
  subject: string;
  text: string;
  templateRevision: string;
};

interface EmailTransport {
  send(email: OutboundEmail): Promise<{ receipt: string }>;
}

const queueAddress: Record<Queue, string> = {
  "orders-eu": "orders-eu@example.test",
  "orders-us": "orders-us@example.test",
  seller: "seller-support@example.test",
  safety: "safety@example.test",
  triage: "triage@example.test",
};

function selectQueue(input: ContactRequest): Queue {
  if (input.category === "safety") return "safety";
  if (input.category === "seller") return "seller";
  if (input.category === "order") {
    return input.market === "EU" ? "orders-eu" : "orders-us";
  }
  return "triage";
}

function renderContactEmail(input: ContactRequest): OutboundEmail {
  const queue = selectQueue(input);
  return {
    messageKey: `contact:${input.id}`,
    to: queueAddress[queue],
    replyTo: input.replyTo,
    subject: `[${queue}] Marketplace contact ${input.id}`,
    text: input.message,
    templateRevision: "contact-v3",
  };
}
```

The types make two decisions obvious. There are exactly five internal destinations, and an unknown category falls into `triage`. There is no provider template ID in the business object. Changing delivery services touches the adapter; changing queue behavior touches the tested business function.

Test the boundary at three levels. Unit tests should cover every category and both markets. Snapshot or fixture tests should verify the rendered subject and text for `contact-v3`, including line breaks and hostile input. A transport contract test should assert the adapter's handling of acceptance, rejection, timeout, and a repeated `messageKey` without sending live customer data.

Before each weekly release, send synthetic messages to controlled EU and US test inboxes, then confirm that the stored receipt and chosen queue match the test case. This is not a deliverability benchmark. It is a release check for the behavior the application owns.

## When remote templates are the better runner-up

Provider-rendered templates are reasonable when a support or lifecycle team changes copy independently of application releases and has a real approval process. They can also suit highly localized content when translators need a dedicated workflow. In those cases, store the remote template identifier and expected revision beside each outbox item, and test that every deployed identifier exists before traffic reaches it.

Repository rendering is not a fit when authorized non-developers must publish urgent copy without a code deployment. It is also limited when localization review already happens in a separate content system. In either case, remote ownership is the better runner-up, provided revision pinning and approval are part of the contract.

Split ownership can work when code owns routing and required fields while a remote system owns a stable layout. The cost is coordination. Define which side controls the subject, legal footer, links, and localization fallback. Without that contract, every incident begins with a search through two systems.

For a solo operator shipping weekly, I would pay that coordination cost only after repository edits become the demonstrated bottleneck. The revenue-per-hour test is blunt: does remote editing release more useful founder time than its second deployment surface consumes? Until the answer is supported by the team's actual workflow, the smaller system wins.

EU and US operation does not change this ownership logic. It adds due-diligence questions for any delivery contract: where message content and metadata are processed, how long they are retained, which subprocessors receive them, and how deletion or access requests are handled. Record the answers. Do not infer them from a region selector or marketing page.

The final selection exercise should therefore use the same five-case fixture against every candidate API: one EU order question, one US order question, one seller request, one safety report, and one unknown category. Run it once with normal responses, once with a transport timeout, and once with the same message key repeated. Measure operator work, not brochure breadth. Can you trace the message key from the stored outbox item to the transport receipt? Can you distinguish permanent rejection from a timeout without reading message content? Does a retry preserve the logical identity instead of producing a second support ticket? Can you export the evidence needed for support? These checks expose the operational cost that an easy signup screen cannot. If two candidates pass, the simpler contract is the better fit. Price can break a tie, but it should not define the architecture.

**Keep the changing business rule close, and rent the delivery machinery.** That division gives a small marketplace a testable queue decision today without welding its templates to an external control plane tomorrow.

## Sources

- RFC 7208, Sender Policy Framework (SPF): https://datatracker.ietf.org/doc/html/rfc7208
- NIST SP 800-63B, Digital Identity Guidelines: https://pages.nist.gov/800-63-3/sp800-63b.html
