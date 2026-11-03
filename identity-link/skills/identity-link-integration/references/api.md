# Identity Link public API — integration reference

Base URL: `https://api.portal.vu-one.com/api/v1` (read it from `IDENTITY_LINK_API_URL`). The
previous host, `api.identity.vu-one.com`, still works. Interactive docs:
`https://api.portal.vu-one.com/api-docs/`.

Every request sends `User-Agent: identity-link-skill/<version>`, with `metadata.version` from the
header of `SKILL.md`. Error bodies are
`{ "error": "<message>", "code"?: "<stable code>" }`, except the token endpoint and the `409` of the
link endpoints (below).

## Token — `POST /oauth/token`

- Auth: HTTP Basic, `base64(clientId:clientSecret)`.
- Body: `grant_type=client_credentials`, as `application/x-www-form-urlencoded` or JSON.
- `200`: `{ "access_token": "<jwt>", "expires_in": 3600, "token_type": "Bearer" }`.
- `401`: `{ "error": "invalid_client", "error_description": "client authentication failed" }`:
  wrong or rotated credentials.
- `501`: `{ "error": "unsupported_grant_type" }`: only `client_credentials` exists.

Rotating credentials in the portal (**Settings → Developers → Rotate Client Id and Secret**)
invalidates the old pair immediately. Rotating the webhook secret (same page) works the same way:
there is a single secret and no overlap window, so events signed with the new one fail
verification until the new value is deployed. Deploy right after rotating, and rely on the
reconciliation to recover what failed in between.

## Create a link — `POST /links`

```json
{
  "type": "FULL FLOW",
  "externalUserId": "user-1234",
  "metadata": { "orderId": "A-998" },
  "callbackUrl": "https://example.com/verification-done"
}
```

| Field | Required | Notes |
| --- | --- | --- |
| `type` | yes | `FULL FLOW` (document and selfie) or `ID ONLY` |
| `externalUserId` | no | Your id for the person. Returned in every webhook. With it, the person is not asked for an email |
| `metadata` | no | Any JSON object. Returned in every webhook, as a JSON string |
| `callbackUrl` | no | Browser redirect when the flow ends. Carries no data |
| `adaptiveFlow` | no | `true`/`false` overrides the account's "face for returning customers" setting for this link. Needs `externalUserId` |

Any other field is rejected with `400`.

- `201`: `{ "linkId", "linkUrl", "verificationId" }`. Share `linkUrl` with the person. Headers, read
  before this link (creating a link consumes nothing):
  - `Verification-Quota-Limit`: verifications the plan allows in the period;
  - `Verification-Quota-Used`: verifications finished, or finishing, in the period;
  - `Verification-Quota-Remaining`: verifications that can still finish in the period;
  - `Verification-Quota-Period-End`: when the period ends, an ISO 8601 timestamp, not seconds (the
    1st of next month on Free). A paid plan that does not renew expires then (`LinkErrorPlanExpired`).
- `409`: quota exhausted (`LinkErrorMaxReached`) or paid plan expired (`LinkErrorPlanExpired`).
  Error bodies here are `{ "code": 409, "name", "description", "data" }`. For `LinkErrorMaxReached`,
  `data` carries `quota: { limit, used, remaining, periodEnd }`, the same values as the headers, and `upgradeUrl`, the portal page to buy a plan:

  ```json
  {
    "code": 409,
    "name": "LinkErrorMaxReached",
    "description": "Max verifications reached",
    "data": {
      "name": "Acme", "color": "#666666", "logo": null,
      "quota": { "limit": 10, "used": 10, "remaining": 0, "periodEnd": "2026-11-01T00:00:00.000Z" },
      "upgradeUrl": "https://portal.vu-one.com/plans"
    }
  }
  ```

The link expires after the account's link window (30 minutes on the free plan). An expired link
cannot be completed: create a new one.

## Read a verification — `GET /verifications/{id}`

- `200`: the record. Main fields:
  - `_id`, `linkId`, `status`: `pending` | `started` | `success` | `failed` | `expired`;
  - `step`: `NEW_OPERATION` | `ADD_FRONT` | `ADD_BACK` | `END_OPERATION`;
  - `data`: what the document and the biometrics returned;
  - `detectedDocumentCountry`, `geolocation`, `createdAt`, `updatedAt`, `expiresAt`.
- `404`: unknown id, not your account's, or anonymized.
- `410`: expired, with body `{ "_id", "status", "expiresAt" }`. `status: "expired"` means the
  person never completed it in time; `success` or `failed` means it finished and only its details
  are gone, because the data window passed (30 minutes after the verification ends on the free
  plan). Keep that final status, except over a stored `failed`: the `410` drops `feedback`, so a
  failure marked after the fact reads `success` there.
- A verification the account marked as failed after the fact keeps `status: "success"` and gains
  `feedback.status: "failed"`: treat it as failed.

## Webhooks

Register once: `POST /webhook-endpoints` with `{ "url": "https://…", "description": "…" }` →
`201 { "_id", "url", "description", "createdAt", "updatedAt" }`. List, read, update and delete
with `GET /webhook-endpoints`, `GET|PATCH|DELETE /webhook-endpoints/{id}`.

`POST` does not deduplicate, and every registered endpoint receives every event: registering the
same URL on each start sends each event that many times. List first and register only when the
URL is missing. Delete an endpoint you no longer use, such as a tunnel URL after a test: it keeps
receiving every event.

Every registered endpoint receives a `POST` with `Content-Type: application/json;charset=UTF-8`
and the header `X-Signature-SHA256`, for these events:

- `verification.started`
- `verification.addfront.started`, `verification.addfront.successful`, `verification.addfront.failed`
- `verification.addback.started`, `verification.addback.successful`, `verification.addback.failed`
- `verification.register.started`, `verification.register.successful`, `verification.register.failed`
- terminal: `verification.successful`, `verification.failed`

Body:

```json
{
  "type": "verification.successful",
  "data": {
    "_verificationId": "43c3ad3a-aa6c-4e4a-b133-c0e1897b55e7",
    "_externalUserId": "user-1234",
    "_metadata": "{\"orderId\":\"A-998\"}",
    "geolocation": { "country": "AR", "city": "Buenos Aires", "source": "ip", "tier": "coarse" }
  }
}
```

- `_metadata` is a JSON string when the link had metadata (`JSON.parse` it), and the object `{}`
  when it had none.
- `geolocation` is only on the two terminal events, and only when one was captured.
- When the account marks a verification as failed after the fact, a `verification.failed` arrives
  with `_statusOverride: true` and a `_reasonId`. The record's `status` stays `success` (see
  `feedback.status` above), so apply the failure from the event.
- No event is sent when a link expires unused: only reading the verification shows it.

**Signature.** `X-Signature-SHA256` is `hex(HMAC-SHA256(webhookSecret, rawBody))`. Verify it over
the raw bytes before parsing: re-serializing a parsed body can change it and break the check. In
Express, register the webhook route with `express.raw()` before any global `express.json()`.

**Delivery.** Up to 3 attempts, 100 ms apart, until the endpoint answers 200, 201, 202 or 204.
After that the event is dropped; there is no later redelivery. Answer fast and do slow work
afterwards. To recover a missed result, read `GET /verifications/{id}` within the data window.

## Quota (free plan)

- 10 finished verifications per calendar month, shared between portal and API, renewed on the 1st.
  Links that are created but never finished do not count.
- Exhausted quota: `POST /links` → `409` with the quota and `upgradeUrl`; opening a link → `409`
  with the company branding only; a capture in progress → `403` with
  `code: "verificationQuotaExceeded"`.
- At 80% of the quota the account owner gets an email, once per period.

## Reference implementation (Node.js 18+, TypeScript, Express)

```ts
import crypto from 'node:crypto';
import express from 'express';

const BASE = process.env.IDENTITY_LINK_API_URL ?? 'https://api.portal.vu-one.com/api/v1';
// metadata.version from the header of SKILL.md, written in as a literal.
const SKILL_VERSION = '<version>';
const USER_AGENT = `identity-link-skill/${SKILL_VERSION} node/${process.versions.node.split('.')[0]}`;
// An empty webhook secret would let anyone sign events: refuse to start without every value.
function requireEnv(name: string): string {
  const value = process.env[name]?.trim();
  if (!value) throw new Error(`${name} is missing or empty`);
  return value;
}

const CLIENT_ID = requireEnv('IDENTITY_LINK_CLIENT_ID');
const CLIENT_SECRET = requireEnv('IDENTITY_LINK_CLIENT_SECRET');
const WEBHOOK_SECRET = requireEnv('IDENTITY_LINK_WEBHOOK_SECRET');

let cached: { token: string; expiresAt: number } | undefined;

async function getToken(): Promise<string> {
  if (cached && Date.now() < cached.expiresAt - 60_000) return cached.token;
  const res = await fetch(`${BASE}/oauth/token`, {
    method: 'POST',
    headers: {
      Authorization: `Basic ${Buffer.from(`${CLIENT_ID}:${CLIENT_SECRET}`).toString('base64')}`,
      'Content-Type': 'application/x-www-form-urlencoded',
      'User-Agent': USER_AGENT,
    },
    body: 'grant_type=client_credentials',
  });
  if (!res.ok) throw new Error(`Identity Link token: HTTP ${res.status}`);
  const body = (await res.json()) as { access_token: string; expires_in: number };
  cached = { token: body.access_token, expiresAt: Date.now() + body.expires_in * 1000 };
  return cached.token;
}

async function api(path: string, init: RequestInit = {}) {
  return fetch(`${BASE}${path}`, {
    ...init,
    headers: {
      Authorization: `Bearer ${await getToken()}`,
      'Content-Type': 'application/json',
      'User-Agent': USER_AGENT,
      ...init.headers,
    },
  });
}

// Quota exhausted. The renewal date and the purchase link are properties, not part of the message:
// an app that answers err.message would otherwise show "buy more" to the person being verified.
export class QuotaExhaustedError extends Error {
  constructor(readonly periodEnd?: string, readonly upgradeUrl?: string) {
    super('Identity Link verification quota exhausted');
    this.name = 'QuotaExhaustedError';
  }
}

// externalUserId comes from the server-side session, never from the client's request.
export async function createVerificationLink(externalUserId: string, metadata?: object) {
  const res = await api('/links', {
    method: 'POST',
    body: JSON.stringify({ type: 'FULL FLOW', externalUserId, metadata }),
  });
  if (res.status === 409) {
    const { name, data } = (await res.json()) as {
      name: string;
      data?: { quota?: { periodEnd: string }; upgradeUrl?: string };
    };
    if (name === 'LinkErrorMaxReached') throw new QuotaExhaustedError(data?.quota?.periodEnd, data?.upgradeUrl);
    throw new Error(`Identity Link create link refused: ${name}`);
  }
  if (!res.ok) throw new Error(`Identity Link create link: HTTP ${res.status}`);
  // Absent on a response that does not report the quota: Number(null) would read as 0 left.
  const remainingHeader = res.headers.get('Verification-Quota-Remaining');
  const remaining = remainingHeader === null ? NaN : Number(remainingHeader);
  if (remaining <= 2) console.warn(`Identity Link: ${remaining} verifications left until ${res.headers.get('Verification-Quota-Period-End')}`);
  return (await res.json()) as { linkId: string; linkUrl: string; verificationId: string };
}

type Outcome = 'pending' | 'success' | 'failed' | 'expired';

// The final status of a verification. A 410 still carries it; only the details are gone.
export async function getOutcome(id: string): Promise<{ outcome: Outcome; record?: any }> {
  const res = await api(`/verifications/${encodeURIComponent(id)}`);
  if (res.status === 410) {
    const { status } = (await res.json()) as { status: string };
    return { outcome: status === 'success' || status === 'failed' ? status : 'expired' };
  }
  if (!res.ok) throw new Error(`Identity Link read verification: HTTP ${res.status}`);
  const record = await res.json();
  // A failure marked by the account after the fact keeps status "success".
  if (record.feedback?.status === 'failed') return { outcome: 'failed', record };
  if (record.status === 'success' || record.status === 'failed' || record.status === 'expired')
    return { outcome: record.status, record };
  return { outcome: 'pending', record };
}

// Every registered endpoint gets every event, so register the URL only once.
export async function ensureWebhookEndpoint(url: string) {
  const res = await api('/webhook-endpoints');
  if (!res.ok) throw new Error(`Identity Link list webhooks: HTTP ${res.status}`);
  const endpoints = (await res.json()) as { url: string }[];
  if (endpoints.some((e) => e.url === url)) return;
  const created = await api('/webhook-endpoints', {
    method: 'POST',
    body: JSON.stringify({ url, description: 'Identity Link results' }),
  });
  if (!created.ok) throw new Error(`Identity Link register webhook: HTTP ${created.status}`);
}

// A stored "failed" is final: after the data window a 410 reads "success" without the feedback
// that marked it failed, so a late read or event must not undo it.
function nextOutcome(stored: Outcome | undefined, incoming: Outcome): Outcome {
  return stored === 'failed' ? 'failed' : incoming;
}

function validSignature(rawBody: Buffer, header: string | undefined): boolean {
  if (!header) return false;
  const expected = crypto.createHmac('sha256', WEBHOOK_SECRET).update(rawBody).digest('hex');
  const a = Buffer.from(expected, 'hex');
  const b = Buffer.from(header, 'hex');
  return a.length === b.length && crypto.timingSafeEqual(a, b);
}

const app = express();

// Raw body: the signature is computed over the exact bytes Identity Link sent. Register this route
// before any global express.json(), or the body arrives parsed and the check fails.
app.post('/webhooks/identity-link', express.raw({ type: 'application/json' }), (req, res) => {
  if (!validSignature(req.body, req.header('X-Signature-SHA256'))) return res.sendStatus(401);
  res.sendStatus(200);

  const event = JSON.parse(req.body.toString('utf8'));
  // A string when the link had metadata, the object {} when it had none.
  const raw = event.data._metadata;
  const metadata = typeof raw === 'string' ? JSON.parse(raw) : raw ?? {};
  if (event.data._statusOverride) {
    // Marked as failed by the account after the fact: the record still says "success".
    // Store "failed" against event.data._externalUserId / metadata.
    return;
  }
  if (event.type === 'verification.successful' || event.type === 'verification.failed') {
    // Read now: the data window is short (30 minutes on the free plan).
    getOutcome(event.data._verificationId)
      .then(({ outcome, record }) => {
        // Store nextOutcome(stored, outcome), and only the fields of record you need, against
        // event.data._externalUserId / metadata. Do not log the record or the webhook body.
      })
      .catch((err) => console.error('Identity Link result', err instanceof Error ? err.message : err));
  }
});

app.listen(3000);
```
