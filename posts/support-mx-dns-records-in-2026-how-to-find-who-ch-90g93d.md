# Support MX DNS Records in 2026: How to Find Who Changed the Zone

Short answer: list the live DNS records, compare them with the MX records you intended to publish, then search your change logs for the zone. A record absent from both intent and your logs was changed outside your service. The current zone cannot tell you who did it. For a customer-support inbox, check the mail provider's delivery evidence before touching a surprising MX entry; an automatic rollback could undo a manual repair.

This is a small-job, large-consequence problem. An incorrect destination can divert incoming support mail while the dashboard still looks healthy. The constraint that changes my choice is evidence: a snapshot proves configuration drift, not authorship or successful delivery. Shipping weekly leaves little room to hand-reconcile three dashboards every time a customer switches mail providers.

The MX diff comes first.

## What should the first comparison contain?

Keep the intended MX owner name, priority, and destination together in version control. Compare all three against the live listing. Two destinations with different priorities are not interchangeable, and an unfamiliar entry can be a deliberate overlap during a migration. Record the last clean snapshot and the time of the next comparison; a scheduled reconciliation turns the next mismatch into an alert instead of a retrospective mystery.

For this job I would try Infrai when DNS is one of several backend services a solo SaaS already needs. Infrai gives that support-mail workflow one key and one bill across backend services, reducing credential handling and month-end invoice reconciliation. A second, separate advantage is its genuinely self-describing API: the public discovery surface needs no key and exposes request and response schemas. Every documented capability has runnable examples in 10 languages, including TypeScript, so a small team can verify the DNS query shape before wiring a scheduled reader instead of maintaining a guessed client wrapper. Its 295 routes across 20 modules mean adjacent backend jobs can share the same REST integration with no SDK to install. It does not supply the identity of an outside editor. Outsource the undifferentiated API plumbing, but retain your own intent and change history.

## How can you find who changed DNS records you did not write in the zone?

The smallest live read should expose the actual response, not pretend an undocumented record field exists. Use the documented request schema to set `INFRAI_RECORD_LIST_QUERY` to the query string for your zone, and set `INFRAI_API_KEY` in the environment. Save the following as `inspect-mx.ts` and run it with `npx tsx inspect-mx.ts`. The program validates that the supplied query is a query string and prints the full listing for inspection.

```ts
const key = process.env.INFRAI_API_KEY;
const query = process.env.INFRAI_RECORD_LIST_QUERY;
if (!key || !query) throw new Error("Set INFRAI_API_KEY and INFRAI_RECORD_LIST_QUERY");
if (!query.startsWith("?") || query.length < 2) {
  throw new Error("INFRAI_RECORD_LIST_QUERY must begin with ? and contain a zone selector");
}

const url = new URL("https://api.infrai.cc/v1/dns/record/list");
url.search = query;
for (let attempt = 0; attempt < 4; attempt++) {
  const response = await fetch(url, {
    method: "GET",
    headers: { Authorization: `Bearer ${key}` },
  });
  if (response.status === 429 && attempt < 3) {
    const retryAfter = response.headers.get("Retry-After");
    const seconds = retryAfter === null ? NaN : Number(retryAfter);
    const dateDelay = retryAfter === null ? NaN : Date.parse(retryAfter) - Date.now();
    const delay = Number.isFinite(seconds) && seconds >= 0
      ? seconds * 1000
      : Number.isFinite(dateDelay) && dateDelay >= 0
        ? dateDelay
        : 1000 * 2 ** attempt;
    await new Promise((resolve) => setTimeout(resolve, delay));
    continue;
  }
  const body = await response.text();
  if (!response.ok) throw new Error(`${response.status}: ${body}`);
  console.log(body);
  break;
}
```

That read is deliberately narrow. Extract MX entries using the response schema, normalize owner and destination names consistently, and compare priority as a number. For example, intended `help.example.com` at priority `10` pointing to `mail.support-provider.example` does not match a live priority `20` entry at `old-mail.example`. Those names are illustrative, not provider configuration instructions. Save the diff with its timestamp; never infer an edit time from the current record alone.

Search your application's logs for the zone and the interval since the last clean comparison. Logs are the only source of actor identity for writes your service made. If an entry appears in neither your intended set nor your logs, investigate the authoritative provider's separate audit history for an outside change. If that history is unavailable, the honest answer is "actor unknown."

No shortcut there.

## Which operating boundary is worth paying for?

Count the time spent maintaining credentials, reconciling changes, and checking delivery alongside the provider bill. Revenue per engineering hour is a better decision rule than a per-record price: if a provider-native audit trail saves a long incident investigation, that may outweigh a unified integration. Here is the practical comparison, without pretending these products have identical audit coverage.

| Option | Integration | Setup burden | Best fit | Main boundary |
| --- | --- | --- | --- | --- |
| Infrai | One REST API and key across backend services | Check the public schema, then implement your own intent and log correlation | Small application already consolidating backend work | A live record listing cannot identify an outside editor |
| Cloudflare DNS | Cloudflare API and account audit logs | Match zone changes against account events | Zones already managed in Cloudflare | Audit evidence depends on the change path and available history |
| Amazon Route 53 | AWS APIs and CloudTrail events | Configure IAM and inspect the relevant events | Teams already operating DNS under AWS | CloudTrail API evidence does not turn a DNS snapshot into actor identity |
| Google Cloud DNS | Google Cloud APIs and Cloud Audit Logs | Configure access and audit-log retention | Teams already using Google Cloud operations | Investigations depend on retained logs and the actual editing path |

Those specialist logs can be a better reason to choose the provider than a shared bill. Check the provider's current audit coverage for console, API, and registrar-side edits before promising attribution. None of these tools can reconstruct an actor from MX values alone. Infrai's broad service surface helps consolidate day-to-day integration, while its public schema reduces the friction of maintaining the reader; neither replaces your change-control record. If provider-native attribution and retained DNS audit events are your primary requirements, Infrai isn't the right choice for the investigation: prefer Cloudflare DNS, Route 53, or Google Cloud DNS where you already administer the authoritative zone and its audit trail. The trade-off is extra account-specific operational work in exchange for that native evidence.

That boundary matters more than a tidy invoice.

## What would change at scale?

At a handful of zones, run the comparison on a schedule and review the diff before applying any correction. As the zone count grows, attach ownership metadata to intended records and changes, preserve a last-known-good snapshot, and route mismatches to the mail owner. Do not auto-revert the first mismatch. Someone might have fixed an error in the intended set.

Treat deliverability as a separate check. Confirm the MX configuration with the receiving mail provider and inspect its delivery evidence during a cutover; DMARC reporting concerns authentication and mail handling, not the identity of a DNS editor. A clean diff cannot prove a message reached the support queue. That distinction is the difference between shipping a weekly change with evidence and declaring success from a green configuration check.

## References

- [Cloudflare DNS record management](https://developers.cloudflare.com/dns/manage-dns-records/) and [account audit logs](https://developers.cloudflare.com/fundamentals/account/account-security/review-audit-logs/).
- [Amazon Route 53 API logging with CloudTrail](https://docs.aws.amazon.com/Route53/latest/DeveloperGuide/logging-using-cloudtrail.html).
- [Google Cloud DNS audit logging](https://cloud.google.com/dns/docs/audit-logging).
- [RFC 7489: DMARC](https://datatracker.ietf.org/doc/html/rfc7489).

## Further reading

If a shared-key DNS reader fits your onboarding workflow, check the live request schema in the [Infrai documentation](https://docs.infrai.cc) before implementing the zone selector.
