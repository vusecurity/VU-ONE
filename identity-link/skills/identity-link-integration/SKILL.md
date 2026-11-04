---
name: identity-link-integration
description: Integrate VU Identity Link identity verification into an application's
  code through its public API (OAuth client credentials, verification links, signed
  webhooks, verification results). Use when the user wants their software to verify
  the identity of their own users with Identity Link (also when they just say "VU"
  for this), create verification links, receive and validate Identity Link webhooks,
  read or recover a verification result, handle its API errors, or build a quick proof
  of concept on the Identity Link free plan. Not for VU Cloud Services (the Api-Key
  APIs such as ID Analysis, Face Analysis or Facial Antispoofing), VU ONE login or
  tenants, or using the Identity Link portal without writing code.
metadata:
  version: 1.0.1
  updated: '2026-11-04'
---

# Identity Link integration

Identity Link verifies a person's identity (ID document plus a selfie with liveness) through a link
the integrator shares with that person. This skill takes a developer from no account to a working
integration in their own codebase: get credentials, request a token, create a link, receive the
webhook, verify its signature, and read the result.

Write the integration in the user's language and framework. When nothing is specified, use Node.js
with TypeScript. The full API details, payloads and a reference implementation are in
[references/api.md](references/api.md): read it before writing code.

## How the flow works

1. The integrator's backend gets an access token with its client credentials.
2. It creates a verification link and gets back `linkUrl`, `linkId` and `verificationId`.
3. It sends `linkUrl` to the person being verified, who opens it in a browser or phone and captures
   their document and a selfie.
4. Identity Link calls the integrator's webhook endpoint as the verification progresses, and once
   more when it ends (`verification.successful` or `verification.failed`).
5. The backend verifies the webhook signature and reads the verification result.

The `callbackUrl` of a link only redirects the person's browser when the flow ends, with no data.
**The result reaches the backend through the webhook**, never through the redirect.

## Server and client

Everything in this skill runs on the integrator's server: the credentials and webhook secret, the
token, `POST /links`, the webhook handler and `GET /verifications`. The browser or mobile app only
receives `linkUrl` and opens it. In a SPA, Next.js or mobile app, put these calls in a backend or
server route, never in client code: the "client" of client credentials is the integrator's
application, not the browser, and a secret in a bundle or APK is public.

## Steps

### 1. Account and credentials

- The user needs an Identity Link account. If they have none, tell them to sign up at
  `https://portal.vu-one.com`: the free plan is enough for a proof of concept.
- In the portal, **Settings → Developers** shows the `Client Id`, `Client Secret` and
  `Webhook Secret`. Ask the user to put them in environment variables, never in code or chat:
  `IDENTITY_LINK_CLIENT_ID`, `IDENTITY_LINK_CLIENT_SECRET`, `IDENTITY_LINK_WEBHOOK_SECRET`.
- Check that `.env` is in `.gitignore` before writing one. In production, keep the values in a
  secret manager (Key Vault, Secrets Manager, Vault or the platform's own) and inject them as
  environment variables. Do not read or print the values yourself. If one leaks, rotate it in
  **Settings → Developers**: the old value stops working at once (see
  [references/api.md](references/api.md)).
- Refuse to start when any of them is missing or empty: an empty webhook secret is an HMAC key
  anyone can use to forge events.
- The API base URL is `https://api.portal.vu-one.com/api/v1`. Read it from
  `IDENTITY_LINK_API_URL` with that default, so another environment can be used without code
  changes.

### 2. Token

- `POST {base}/oauth/token` with HTTP Basic auth (`clientId:clientSecret`) and
  `grant_type=client_credentials`. It returns `access_token` and `expires_in` (seconds).
- Cache the token and request a new one shortly before it expires. Do not request one per call.

### 3. Link

- `POST {base}/links` with `Authorization: Bearer <token>` and at least `{ "type": "FULL FLOW" }`.
- Send `externalUserId` (the integrator's id for the person) and `metadata` (any JSON object):
  both come back in every webhook, which is how the integrator matches an event to its own user.
- Take `externalUserId` from the server-side session, never from the request body or URL: if the
  client chooses it, an attacker creates a link with the victim's id, verifies with their own
  document, and leaves the victim's account verified.
- Keep `externalUserId` and `metadata` opaque (internal ids, no email or other personal data):
  they travel in every webhook.
- Store `verificationId` and `linkId`, then give `linkUrl` to the person.
- A link expires (30 minutes on the free plan) and cannot be completed afterwards. When the person
  asks again later, create a new link instead of resending the old one.

### 4. Webhook

- Register the endpoint once: `POST {base}/webhook-endpoints` with `{ "url", "description" }`.
  Every registered endpoint receives every event and `POST` does not deduplicate, so list them
  first (`GET {base}/webhook-endpoints`) and register only when the URL is missing: registering on
  each start delivers every event several times.
  It must be reachable from the internet; for local development use a tunnel (ngrok, cloudflared)
  and register the tunnel URL. When the test ends, delete that endpoint
  (`DELETE {base}/webhook-endpoints/{id}`): it keeps receiving every event, with personal data,
  and tunnel subdomains get reassigned to other people.
- The handler must **verify the `X-Signature-SHA256` header before trusting anything**: an
  HMAC-SHA256, hex encoded, of the **raw request body**, keyed with the webhook secret. Compute it
  over the exact bytes received, before any JSON parsing, and compare in constant time.
  In Express, mount the webhook route with `express.raw()` **before** any global `express.json()`:
  otherwise the body arrives already parsed and the check fails. Fix the order, never the check.
- Answer `200` quickly. A non-2xx answer is retried only twice more, 100 ms apart, then the event
  is lost: do any slow work after answering.
- Identify the verification with `data._verificationId`. `data._metadata` arrives as a JSON
  **string** when the link had metadata (parse it) and as `{}` when it had none.

### 5. Result

- On `verification.successful` or `verification.failed`, call `GET {base}/verifications/{id}` to
  read the full record (`status`, `data`, `geolocation`, …).
- **Read it right away.** The result is kept only for the plan's data window: on the free plan,
  30 minutes after the verification ends. After that the API answers `410`.
- When the account marks a verification as failed after the fact, a `verification.failed` arrives
  with `_statusOverride: true`, but the record keeps `status: "success"` and only gains
  `feedback.status: "failed"`. Treat that as failed, from the event or from `feedback.status`.
- **A failed verification stays failed.** Once the data window passes, the `410` says
  `status: "success"` with no `feedback`, so a later read or a late event would undo a failure
  marked after the fact. Never move a stored `failed` to `success`.
- `success` proves that a real person completed the flow, not that it is the integrator's user: a
  link can be forwarded. Where the integrator already knows the person, recommend comparing `data`
  (name, document number, date of birth) with what it has.
- Store only what the integrator needs. Do not log webhook bodies, records or `linkUrl` (whoever
  has the link can complete the verification).

### 6. Do not lose results

The webhook is the only push, and it is not durable: if the integrator's server is down or slow
for those three attempts, that event is gone and never sent again. A production integration needs
a fallback, even when nobody asked for one:

- Persist the open verifications (`verificationId` → the integrator's user) somewhere that
  survives a restart.
- Reconcile them: call `GET {base}/verifications/{id}` for each open one, **at startup and on a
  schedule shorter than the data window** (every few minutes; never 30 minutes or more on the free
  plan), and apply any `success` or `failed` the webhook did not deliver.
- A `410` still carries the final `status`: `success` or `failed` means it finished and only the
  details are gone, so apply that status (never over a stored `failed`); `expired` means the person never completed it, so close
  it and, if needed, start a new one. A link that expires unused sends no webhook: only this read
  shows it. Do not let one expired record stop the reconciliation of the others.

## Identify the integration

Read `metadata.version` from this skill’s frontmatter and use that value in
`User-Agent: identity-link-skill/<version>` on **every** request to the Identity Link API.
Replace `<version>` with the actual version when generating code; optionally append the runtime. It lets VU tell
integrations built with this skill apart from others. It carries no personal data.

## Free plan limits to explain to the user

- **10 verifications per calendar month**, shared between the portal and the API, renewed on the
  1st. Only finished verifications count; creating a link does not.
- Every `201` of `POST /links` carries `Verification-Quota-Limit`, `Verification-Quota-Used`,
  `Verification-Quota-Remaining` and `Verification-Quota-Period-End` (an ISO 8601 timestamp, not
  seconds). A missing header means "unknown", never 0. Read them and warn the account's administrators (a log line or
  an internal alert, never the end user) when few remain, so they can buy more before the cut.
- When the quota is used up, `POST /links` and opening a link answer `409`, and a capture already
  in progress is refused with `403` (`code: verificationQuotaExceeded`). Handle these as "quota
  exhausted", not as generic errors. The `409` of `POST /links` carries `data.quota.periodEnd` and
  `data.upgradeUrl`: log both, and show them to whoever runs the account, so the error says when it
  renews and where to buy more. The person being verified only needs a "try again later".
- The account owner also gets an email at 80% of the quota, once per period.
- Branding, link and data expiration times, and location capture are paid settings: the free plan
  keeps fixed values (30 minutes for the link and for the data).

## Before you finish

- [ ] Every Identity Link call and secret lives on the server; the client only gets `linkUrl`.
- [ ] Credentials and the webhook secret come from environment variables (`.env` git-ignored), and
      the app refuses to start when one is missing or empty.
- [ ] `externalUserId` comes from the server-side session, and it and `metadata` carry no
      personal data.
- [ ] The token is cached and refreshed before `expires_in`.
- [ ] Every request sends the `identity-link-skill/` User-Agent.
- [ ] The webhook signature is verified on the raw body, in constant time, before parsing.
- [ ] The webhook answers 2xx fast and reads the result right after a terminal event.
- [ ] The webhook endpoint is registered once, after checking the existing list.
- [ ] Open verifications are persisted and reconciled at startup and on a schedule shorter than the
      data window, so a missed webhook does not lose a result; a `410` keeps its final status.
- [ ] A failure marked after the fact (`_statusOverride`, `feedback.status: "failed"`) wins over
      `status: "success"`, and a stored `failed` is never moved to `success`.
- [ ] Webhook bodies, records and `linkUrl` stay out of the logs.
- [ ] A webhook endpoint registered for a tunnel is deleted when the test ends.
- [ ] A user asking again after their link expired gets a new link.
- [ ] `externalUserId` (or `metadata`) maps each verification to the integrator's user.
- [ ] Quota errors (409/403) are handled and explained: the `409` logs `periodEnd` and `upgradeUrl`,
      and the quota headers of each created link warn before the cut.
- [ ] Walk the user through an end-to-end test: create a link, open it on a phone, complete it,
      and see the webhook arrive and the result read.
