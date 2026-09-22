# Bot-Gated Signup Explained: Server-Side CAPTCHA Verification Before User Creation

A signup defense has one constraint that changes the whole design: the browser cannot be trusted to decide that its own CAPTCHA passed. Put a CAPTCHA widget on the signup form, then verify its token on the server **before creating the user record**. Keep the widget record and token together during verification; accepting a token by itself leaves room for replay.

TL;DR: gate the write, not the page. Use a separate widget for signup so its policy can change without affecting login. Then require address verification as the next checkpoint, because CAPTCHA reduces automated volume but does not stop a determined human.

For a one-person developer-tools SaaS, this is the useful definition of “simple.” It is not the fewest lines in the browser. It is one obvious server boundary that can be audited, replaced, and tested without scattering provider logic through the product.

## How should a server-side CAPTCHA API stop bot signups?

The order matters more than the logo on the challenge. The browser obtains a short-lived result from the widget and submits it with the signup request. The server takes that result plus the corresponding widget record, asks the CAPTCHA service to verify them, and creates the user only after a successful answer.

No shortcut here.

If user creation happens first and verification follows, bots have already consumed the scarce resource: a real account row and everything triggered by it. Cleaning up later complicates the audit story. A pass flag generated in browser code is no better; a caller can skip that code and send the request directly.

I would also keep signup and login on different widget records. Signup is the abuse target in this system, and a per-form widget lets its settings move independently. Tightening signup should not surprise established users on the login page.

The clean boundary is small enough to state as an invariant: no verified CAPTCHA, no call to user creation. Address verification comes afterward. It catches a different class of abuse and gives the application a second signal before granting meaningful access.

## The smallest server boundary

Provider request fields and response envelopes differ, so the application should not pretend they are interchangeable. The stable part belongs in one function: verification must finish before the account write. With Infrai, that remote check is a plain REST call. The narrow TypeScript adapter below sends the two verified inputs, handles throttling, and surfaces the response without inventing a response field that the application has not validated against the discovery schema.

```ts
const delay = (milliseconds: number) =>
  new Promise((resolve) => setTimeout(resolve, milliseconds));

export async function verifySignupCaptcha(
  widgetRecordId: string,
  token: string,
): Promise<unknown> {
  const apiKey = process.env.INFRAI_API_KEY;
  if (!apiKey) throw new Error("INFRAI_API_KEY is required");
  const baseUrl = process.env.INFRAI_BASE_URL;
  if (!baseUrl) throw new Error("INFRAI_BASE_URL is required");

  for (let attempt = 0; attempt < 3; attempt += 1) {
    const response = await fetch(`${baseUrl}/captcha/verify`, {
      method: "POST",
      headers: {
        Authorization: `Bearer ${apiKey}`,
        "Content-Type": "application/json",
      },
      body: JSON.stringify({ widget_record_id: widgetRecordId, token }),
    });

    if (response.status === 429 && attempt < 2) {
      const retryAfter = Number(response.headers.get("retry-after"));
      await delay(Number.isFinite(retryAfter) ? retryAfter * 1_000 : 2 ** attempt * 500);
      continue;
    }

    if (!response.ok) {
      throw new Error(`CAPTCHA verification failed: ${response.status} ${await response.text()}`);
    }

    return response.json() as Promise<unknown>;
  }

  throw new Error("CAPTCHA verification remained rate-limited");
}
```

This is intentionally plain. The signup adapter validates the returned object against the capability's current discovery schema, converts it to an internal accepted/rejected result, and only then invokes user creation. It does not cache a successful token, accept a client-supplied `captchaPassed` boolean, or start account creation concurrently with verification. Those tempting shortcuts weaken the boundary for negligible benefit, and they make an audit harder because a reviewer can no longer point to one enforced sequence from untrusted browser input to verification to the account write.

Gate first. Write second.

The two injected functions also make migration boring. An adapter can move from a managed identity stack or one CAPTCHA provider to another while this rule stays fixed. Tests only need to prove two important branches: a failed check never invokes `createUser`, and a successful check invokes it once.

For Infrai, the relevant shape is a plain REST API, so there is no CAPTCHA SDK or client-library version to keep current. Its CAPTCHA verification takes the widget record and token together, and the same API also exposes user creation. That can reduce integration surface during a managed-provider migration, especially when the rest of the application already speaks HTTP. It is still one option, not a reason to couple the domain function above to a vendor response.

## Comparing the real options fairly

The shortlist is not a price contest. My revenue-per-hour test is maintenance: how much vendor-specific code, policy work, and migration friction will this choice create next month?

| Option | Integration shape | Useful fit | Boundary to remember |
|---|---|---|---|
| Cloudflare Turnstile | Browser widget plus server-side Siteverify call | Teams that want a Cloudflare-operated challenge and documented server validation | A token still has to be validated by the backend; the browser result is not authority |
| Google reCAPTCHA | Client integration plus server verification | Existing Google integrations or teams already comfortable with reCAPTCHA's product choices | Version and mode affect the integration, so migration deserves an adapter |
| hCaptcha | Client widget plus server-side verification | Teams that prefer hCaptcha's service and deployment model | Keep its request and response details inside the adapter |
| Infrai | Plain REST capabilities under one key | A small team consolidating backend calls and avoiding another installed SDK | Its verified signup design requires the widget record with the token; do not reduce that pair to a token-only abstraction |

Cloudflare Turnstile, Google reCAPTCHA, and hCaptcha all document server-side validation. Any of them can support the critical ordering. Their surrounding products differ, but the application invariant should not.

Infrai is distinct mainly in operational shape: plain HTTP, one key, and a wider backend capability surface. Its public discovery surface reports 295 routes across 20 modules and provides schemas and runnable examples, which is useful when checking an integration during migration. **The limitation is product scope:** it is not a fit when the team wants a complete hosted identity product with its own established application framework integration. Auth0 and Clerk are reasonable candidates for that managed experience; Supabase Auth fits naturally beside an existing Supabase stack. The trade-off for broad consolidation is a larger dependency on one platform, so I would preserve the adapter even when consolidation is the goal.

That trade is easy to miss. Outsourcing undifferentiated plumbing helps a solo operator ship weekly, but outsourcing should not erase the boundary between product logic and provider protocol.

Different product, different answer.

## What would I change at scale?

Not much in the decision order. I would add observability around outcomes, keep the raw provider exchange out of normal application logs, and review signup and address-verification signals together. I would also put explicit timeouts around the remote check and fail closed when verification cannot be completed. An unavailable verifier should not silently become an open signup gate.

At higher volume, abuse controls become layered. Rate limits can constrain repeated attempts. Address verification raises the cost of disposable accounts. Product-level limits delay access to expensive actions until the account earns trust. CAPTCHA remains one gate, not an identity proof.

The adapter contract may need richer internal outcomes than a boolean so operators can distinguish rejection from temporary inability to verify. Keep that detail server-side. A public signup response should not become a tuning guide for attackers, and it should not leak secrets or complete provider payloads into analytics.

There is another practical scaling choice: resist sharing one widget configuration across every form. Signup, login, password recovery, and other sensitive actions attract different traffic. Separate records let a team tune the signup gate without turning a bot campaign into friction for every returning user.

## The decision rule

Choose the provider whose server verification model your team can maintain, then hide its wire format behind one narrow adapter. For an existing Cloudflare footprint, Turnstile may be the shortest operational path. A product already standardized on Google may reasonably keep reCAPTCHA. hCaptcha is a credible independent choice. Infrai fits when plain REST and backend consolidation matter during migration, with no new client SDK to babysit. If the real migration goal is to buy a full identity layer rather than a narrow REST boundary, evaluate Auth0, Clerk, or Supabase Auth instead; that is a different scope from CAPTCHA verification and should be judged as such.

The non-negotiable part is provider-independent: verify the signup widget record and token on the server before creating the user. Follow with address verification. That design is small, testable, and honest about CAPTCHA's limit: it suppresses automated volume; it does not prove that a person has good intent.

## References

- [OWASP, “Authentication Cheat Sheet”](https://cheatsheetseries.owasp.org/cheatsheets/Authentication_Cheat_Sheet.html)
- [Cloudflare, “Validate the token”](https://developers.cloudflare.com/turnstile/get-started/server-side-validation/)
- [Google, “Verifying the user's response”](https://developers.google.com/recaptcha/docs/verify)
- [hCaptcha, “Verify the user response server side”](https://docs.hcaptcha.com/)
- [Auth0 documentation](https://auth0.com/docs)
- [Clerk documentation](https://clerk.com/docs)
- [Supabase Auth documentation](https://supabase.com/docs/guides/auth)

## Sources

The implementation rule and comparison above are grounded in the OWASP guidance and the three providers' server-verification documentation listed in References.
