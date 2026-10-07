# Chapter 7 — The API Reference

> **Day 7 · Goal: every endpoint written, documented, and covered by a contract test.**
>
> This is the lookup document. Chapters 12–15 taught *why* each decision was made; this one is
> the complete surface, so you can implement against it without rereading four chapters.

---

## Understand this first

### A reference is not a tutorial

A tutorial explains reasoning. A reference does not. If this chapter says "returns 409 on
`syncVersion` mismatch" without explaining that it exists to prevent a half-migrated vault, you
should go read Chapter 8 — and then come back.

What this chapter owes you is **completeness and exactness**: every path, every field, every
status code, every error code. A reference that is 90% complete is worse than none, because
you will trust the part that is missing.

### Twelve endpoints. That is the whole API.

Count them: 2 health, 6 auth, 3 device, 4 sync, 3 account. **Eighteen routes, of which twelve
are load-bearing.**

That number is a design constraint, not a coincidence. Every endpoint is a place where a
schema change becomes a migration, an auth rule becomes a bug, and a rate limit becomes an
outage. A sync API with forty endpoints is a sync API nobody can reason about.

> **The strongest review question you can ask about any new endpoint: "which existing endpoint
> could this have been a field on?"** If the answer is "none", you need a very good reason. If
> the answer is "actually, one of them", you just saved a migration.

### Versioning: sync carries its own version, everything else is in the path

```ts
/sync/*   →  SYNC_VERSION in the body. Structural changes bump it. Client re-syncs.
/auth/*   →  no version. Access tokens last 15 minutes. You can change a shape in 15 minutes.
/account  →  no version. Rarely called, rarely changed.
```

You do **not** need `/v1/` on anything yet. Reasoning:

- The sync protocol is the only one with a real compatibility problem, and it solves that
  itself with `SYNC_VERSION` in the payload (Chapter 8).
- Auth shapes are consumed by clients you control. A 15-minute access token means a broken
  auth shape is a 15-minute outage, not a permanent one.
- A `/v2/` path you never exercise is dead code, and dead API versions are how you end up
  maintaining three of them forever.

**Add `/v1/` when you have a second consumer you do not control.** That is the only trigger.

---

## Step 1 — Conventions

### Base URLs

| Environment | Base URL | Data |
|---|---|---|
| Local | `http://localhost:8787` | Local MongoDB |
| Staging | `https://api-staging.refrain.dev` | Atlas `refrain_staging` |
| Production | `https://api.refrain.dev` | Atlas `refrain` |

Staging uses a **separate database and a separate JWT keypair**. Sharing either one with
production means staging tests are not testing production, and a leaked staging token is a
production credential.

### Headers

```http
# Required on everything authenticated
Authorization: Bearer <accessToken>

# Sent on every request. Scrubbed from logs; used for device registration.
X-Device-Label: Rohit's MacBook

# Content type. Required for POST/PATCH/PUT with a body.
Content-Type: application/json

# Optional. Request id, generated client-side for tracing.
X-Request-Id: 01J8XK2M9Q
```

**`X-Request-Id` is generated client-side and returned unchanged.** It appears in every server
log line, so when a user reports "sync failed around 3pm", you can find the exact request
instead of guessing from timestamps. It is the difference between a five-minute support answer
and a two-day one.

### Response envelope

Success returns the resource **directly**, not wrapped:

```jsonc
// GET /sync/pull
{ "entries": [...], "cursor": 412, "hasMore": false, "serverTime": 1757012345678 }
```

Failure returns exactly one shape, always:

```jsonc
{
  "error": {
    "code": "sync_version_mismatch",
    "message": "This client needs a full re-sync.",
    "details": { "requiredSyncVersion": 2 }
  }
}
```

> **One error shape, no exceptions.** A client that handles `error.code` handles every failure
> mode the server will ever produce. Mixed shapes — sometimes `{error}`, sometimes `{message}`,
> sometimes a bare array — are why clients end up with `try { } catch { }` and give up.
>
> `details` is optional and is for machine-readable extras only (`requiredSyncVersion`,
> `fieldErrors`). Never put user-facing text there — that goes in `message`.

### Date and number formats

| Type | Format | Example |
|---|---|---|
| Timestamp | Unix ms, integer | `1757012345678` |
| Date string | ISO 8601, UTC | `"2026-10-05T09:12:34.567Z"` |
| Binary | base64url, no padding | `"kQ8vJ0m..."` |
| ID (server) | 24-char ObjectId hex | `"665f1a2b3c4d5e6f7a8b9c0d"` |
| ID (client) | UUID v4 | `"3f2504e0-4f89-41d3-9a0c-0305e82c3301"` |

**Timestamps are Unix milliseconds, not ISO strings.** ISO strings are unambiguous but bulky
and awkward to compare. Integers sort correctly, diff correctly, and cost 4 bytes instead of
24. `extractedAt` inside `Provenance` is an ISO string because it ships to the UI and gets
read by humans in devtools — that is a different trade in a different place.

**Binary is base64url without padding.** Standard base64 contains `+` and `/`, both of which
need escaping inside a JSON string and inside a URL. base64url is URL- and JSON-safe by
construction.

### Pagination

Two shapes, deliberately different:

```ts
// Cursor-based. For sync, blobs, and anything ordered by a monotonic value.
{ "items": [...], "cursor": "412", "hasMore": true }

// Skip/limit. For lists the user will page through by hand (devices, account activity).
{ "items": [...], "total": 7, "page": 1, "pageSize": 20 }
```

**Never use offset pagination on anything that changes while the user is reading it.** If a
device is revoked between page 1 and page 2, an offset shifts by one and you show a duplicate
and skip a row. Cursor-based is stable under concurrent writes.

**`hasMore` is cheaper than a count.** Counting 10,000 rows on every page is a full scan on
your hot collection. Fetch `limit + 1`, and if you got the extra row, you know there is more.
`syncmeta` is small enough that a count is tempting here — resist it, because it is the same
query on the same collection when you grow.

---

## Step 2 — The error catalogue

**Ten codes. Every error the API can return is one of these.**

| Code | HTTP | Meaning | Client should |
|---|---|---|---|
| `validation_failed` | 422 | Body failed Zod | Show field errors. Do not retry. |
| `unauthorized` | 401 | Missing, invalid, or expired token | Refresh once, then stop. |
| `forbidden` | 403 | Valid token, insufficient rights | Stop. Do not retry. |
| `not_found` | 404 | No such resource *for this user* | Stop. |
| `conflict` | 409 | State mismatch (sync version, base revision) | Act on `details`. |
| `sync_version_mismatch` | 409 | Client protocol version differs | Full re-sync. |
| `rate_limited` | 429 | Too many requests | Back off per `Retry-After`. |
| `payload_too_large` | 413 | Blob over 10MB | Do not retry. Tell the user. |
| `unsupported_media_type` | 415 | Wrong content type | Do not retry. Bug. |
| `internal` | 500 | Unhandled | Retry with backoff, max 2 attempts. |

```ts
// packages/fields/src/api.ts — SHARED. Client and server both import this.
import { z } from "zod"

export const ErrorCode = z.enum([
  "validation_failed", "unauthorized", "forbidden", "not_found",
  "conflict", "sync_version_mismatch", "rate_limited",
  "payload_too_large", "unsupported_media_type", "internal",
])
export type ErrorCode = z.infer<typeof ErrorCode>

export const ApiError = z.object({
  error: z.object({
    code: ErrorCode,
    message: z.string(),
    details: z.record(z.string(), z.unknown()).optional(),
  }),
})
export type ApiError = z.infer<typeof ApiError>

/** The 409 the sync engine needs to distinguish. */
export const SyncVersionMismatch = ApiError.extend({
  error: ApiError.shape.error.extend({
    code: z.literal("sync_version_mismatch"),
    details: z.object({ requiredSyncVersion: z.number().int() }),
  }),
})
```

> **Sharing the error schema through `@refrain/fields` is what makes the API typed end to
> end.** `packages/fields` has zero dependencies and no DOM — Chapter 2's rule. The client
> narrows on `code` and TypeScript tells you which `details` are present:
>
> ```ts
> if (err.code === "sync_version_mismatch") {
>   await fullResync(err.details.requiredSyncVersion)   // typed, not `any`
> }
> ```

### Status codes, and the ones people get wrong

| Situation | Code | Note |
|---|---|---|
| Read succeeded | 200 | |
| Created | 201 | `Location` header points at the resource |
| Deleted | 200 + `{ "deleted": true }` | Not 204 — the client wants confirmation |
| No body needed | 204 | Only for `DELETE /devices/:id` |
| Validation failed | **422, not 400** | 400 means malformed syntax. 422 means well-formed and wrong. |
| Auth failed | 401 + `WWW-Authenticate` | RFC 7235 requires the header |
| Authenticated but not allowed | **403, not 401** | 401 means "log in again", which is wrong and loops |
| Version conflict | 409 | Not 400, not 500 |
| Rate limited | 429 + `Retry-After: 60` | Without the header the client guesses |

> **The 401-vs-403 mistake causes an infinite login loop.** If a revoked device returns 401, the
> client tries to refresh, the refresh also 401s, the client tries to refresh again, forever.
> Revoked must be **403** — "you are who you say you are, and the answer is still no."
>
> Chapter 6's `requireAuth` throws `unauthorized` for a revoked device. **That is a bug.** Fix
> it to `forbidden`, and verify the client stops refreshing rather than looping.

---

## Step 3 — Health

### `GET /health`

Liveness. No auth. Used by the platform's health check.

```jsonc
// 200
{ "ok": true, "ts": 1757012345678 }
```

### `GET /health/ready`

Readiness. Checks the database actually answers.

```jsonc
// 200
{ "ok": true, "db": true, "ts": 1757012345678 }

// 503 — pulled from the load balancer, not an outage page
{ "ok": false, "db": false, "ts": 1757012345678 }
```

```ts
health.get("/health/ready", async (c) => {
  const db = await dbPing().then(() => true).catch(() => false)
  // A DB blip returns 503, which pulls this instance from the
  // load balancer. It does NOT crash the process. Restarting
  // every instance during a brief Atlas failover turns a 5-second
  // blip into a 5-minute outage.
  return c.json({ ok: db, db, ts: Date.now() }, db ? 200 : 503)
})
```

> **Liveness and readiness are different questions and conflating them is a classic outage.**
> Liveness asks "is the process wedged" and should **never** check MongoDB — if the database
> blips, restarting every instance does not fix it and makes things worse. Readiness asks "can
> this instance serve traffic" and should check the database. Two endpoints, two purposes.

---

## Step 4 — Auth

Six endpoints. `k_auth` in, tokens out. **The passphrase never appears in any of these.**

### `POST /auth/register`

```ts
const RegisterRequest = z.object({
  email: z.string().email().max(254),
  /** base64url, 32 bytes, = HKDF(k_root, "refrain:auth:v1"). NEVER the passphrase. */
  authKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/, "must be base64url 32 bytes"),
})
```

```jsonc
// 201
{
  "accessToken": "eyJhbGciOiJFZERTQSIsImtpZCI6...",
  "refreshToken": "8Jm2xK9pQrL4vN7wY1zA0bC3dE5fG6hI...",
  "user": {
    "id": "665f1a2b3c4d5e6f7a8b9c0d",
    "email": "rohit@example.com",
    "plan": "free",
    "createdAt": 1757012345678
  },
  "device": { "id": "665f1a2b3c4d5e6f7a8b9c0e", "label": "MacBook", "platform": "chrome" }
}
```

| Status | When |
|---|---|
| 201 | Created |
| 422 | Failed validation |
| 422 | *"Could not create that account."* — also when the email exists. **Same code, same message.** |
| 429 | Over 5/min |

> **`authKey` has a strict regex.** `43` characters is exactly base64url-encoded 32 bytes with
> padding stripped. This catches a client that sends a passphrase by mistake (variable length),
> base64 with padding, or hex — before it costs an Argon2id computation on garbage input.

```ts
auth.post("/register", limitAuth, rateLimitRefresh,
  zValidator("json", RegisterRequest), async (c) => { /* Chapter 6 Step 3 */ })
```

### `POST /auth/login`

```ts
const LoginRequest = z.object({
  email: z.string().email().max(254),
  authKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/),
})
```

Same 200 shape as register. Same 401 message for unknown-email and wrong-key, **including
timing** — Chapter 6 Step 4.

### `POST /auth/refresh`

```ts
const RefreshRequest = z.object({
  refreshToken: z.string().min(32).max(128),
})
```

```jsonc
// 200
{ "accessToken": "eyJ...", "refreshToken": "NEW_8Jm2xK9pQrL4..." }
```

| Status | When | Client action |
|---|---|---|
| 200 | Rotated | Store the new refresh token. **The old one is dead.** |
| 401 | Missing, expired, or revoked | Stop refreshing. Sign in again. |
| 401 | **Reuse detected** | The device family is revoked. Sign in again. |

> **The new refresh token replaces the old one, always.** There is no grace window where both
> work — that is what makes reuse detection work at all. A client that keeps the old token
> after a successful refresh will present it on the *next* refresh and get logged out. This is
> the single most common client bug in rotation systems, so `SyncSession` overwrites both
> fields together and never keeps a "previous" copy.

### `POST /auth/logout`

```ts
const LogoutRequest = z.object({ refreshToken: z.string().min(32).max(128) })
```

```jsonc
// 200 — revokes this device family only
{ "loggedOut": true, "deviceRevokedAt": 1757012345678 }
```

| Status | When |
|---|---|
| 200 | Revoked (or the token did not exist — same response) |
| 401 | No access token |

> **Logout revokes the device, not the account.** Your phone stays signed in. Chapter 6 Step 8:
> revoking access is not deleting data, and the user should not have to log in everywhere to
> log out of one machine.

### `GET /auth/me`

```jsonc
// 200
{
  "id": "665f1a2b3c4d5e6f7a8b9c0d",
  "email": "rohit@example.com",
  "plan": "free",
  "createdAt": 1757012345678,
  "device": { "id": "...", "label": "MacBook", "platform": "chrome", "lastSeenAt": 1757012345678 }
}
```

> **This is the one endpoint that legitimately returns an email.** The user is authenticated and
> asking about their own account. But notice what it does *not* return: `authHash`, any vault
> key, any document. `publicUser()` is one function used by every route so a future field
> cannot leak by being added to the model but not the projection.

### `PUT /auth/auth-key`

The new credential after a passphrase change.

```ts
const AuthKeyRequest = z.object({
  authKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/),
  /** Re-prove possession of the old credential. */
  previousAuthKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/),
})
```

```jsonc
// 200
{ "authKeyUpdatedAt": 1757012345678 }
```

| Status | When |
|---|---|
| 200 | Updated |
| 401 | `previousAuthKey` did not verify |
| 429 | Over 5/min |

> **`previousAuthKey` is required and it is not ceremony.** Without it, anyone who can reach a
> logged-in session can swap the credential and lock the real user out permanently. With it,
> rotating the credential requires proving you hold the current one. **This endpoint also
> invalidates every other device's refresh token**, because a credential change means the
> account may be compromised.

### `GET /.well-known/jwks.json`

Public. No auth. Ed25519 public key, standard JWKS format.

```jsonc
// 200
{
  "keys": [
    {
      "kid": "ed25519-2026-10",
      "kty": "OKP",
      "crv": "Ed25519",
      "x": "11qYAYLefJHBRLG6Ht1nJ0hbTOGHJYDLHX8sG-Xljr0",
      "alg": "EdDSA",
      "use": "sig"
    }
  ]
}
```

> **`kid` is the key id and it is how rotation works.** When you rotate, the new key gets a new
> `kid` and both are published during the overlap. Clients cache by `kid`, so a rotation is a
> config change rather than an outage. This is why Chapter 6 said "add `kid` from the start."

---

## Step 5 — Devices

### `GET /devices`

```ts
const DeviceListQuery = z.object({
  page: z.coerce.number().int().min(1).default(1),
  pageSize: z.coerce.number().int().min(1).max(50).default(20),
})
```

```jsonc
// 200
{
  "items": [
    {
      "id": "665f1a2b3c4d5e6f7a8b9c0e",
      "label": "Rohit's MacBook",
      "platform": "chrome",
      "createdAt": 1756012345678,
      "lastSeenAt": 1757012345678,
      "isCurrent": true
    }
  ],
  "total": 2, "page": 1, "pageSize": 20
}
```

> **`isCurrent` is computed server-side**, because only the server knows which `did` is in the
> presented token. The client cannot derive it. A page that renders "this device" next to two
> others is a real trust signal — the user is deciding whether to revoke, and "which one is
> me" is the first question.

### `PATCH /devices/:id`

```ts
const DevicePatch = z.object({ label: z.string().min(1).max(60) })
```

```jsonc
// 200
{ "id": "...", "label": "Work laptop" }
```

Renaming only. Nothing else is user-settable — `platform`, `fingerprint`, `revokedAt` are
server-owned, and a PATCH that accepts them is an escalation path.

### `DELETE /devices/:id`

```jsonc
// 204 — no body
```

| Status | When |
|---|---|
| 204 | Revoked |
| 403 | Trying to revoke the device making the request |
| 404 | Not yours |

> **403, not 422, for self-revocation.** "You cannot revoke yourself" is a permission outcome —
> the request is well-formed and you are correctly authenticated, and the answer is still no.
> 422 would tell the client the body was wrong, which it was not.

---

## Step 6 — Sync

Four endpoints. The core of the product's optional infrastructure.

### `GET /sync/pull`

```ts
const PullQuery = z.object({
  since:   z.coerce.number().int().min(0).default(0),
  limit:   z.coerce.number().int().min(1).max(500).default(500),
  docType: z.enum(["fact", "document", "verse", "setlist"]).optional(),
})
```

```jsonc
// 200
{
  "entries": [
    {
      "docId": "email",
      "docType": "fact",
      "revision": 412,
      "deviceId": "665f1a2b3c4d5e6f7a8b9c0f",
      "updatedAt": 1757012345678,
      "deleted": false
    }
  ],
  "cursor": 412,
  "hasMore": false,
  "serverTime": 1757012345678
}
```

**No ciphertext. No filename. No label. No value.** This is the whole point of Chapter 5's
collection split, and it is what makes an unchanged device's pull under 1KB.

`docType` is optional and exists for one case: a device that only cares about facts can pull
`?docType=fact` and avoid learning that a 12MB Aadhaar scan changed. Useful for a mobile client
on a metered connection.

### `POST /sync/blobs`

Fetch the ciphertext for specific `docId`s.

```ts
const BlobsRequest = z.object({
  docIds: z.array(z.string().min(1).max(128)).min(1).max(100),
})
```

```jsonc
// 200
{
  "blobs": [
    {
      "docId": "email",
      "docType": "fact",
      "revision": 412,
      "algo": "AES-256-GCM",
      "kdfVersion": 1,
      "byteLength": 148,
      "iv": "AAAAAAAAAAAAAAAA",
      "wrappedDek": { "ciphertext": "AQIDBA...", "iv": "BBBBBBBBBBBBBBBB" },
      "ciphertext": "3q2+7w=="
    }
  ]
}
```

| Status | When |
|---|---|
| 200 | Returned (may be fewer than requested — deleted) |
| 401 | Expired token |
| 413 | Request body over the limit |

> **A blob you no longer have is simply absent from the response.** Not a 404 per id — that
> would be N round trips for N deletions. Absent + a `deleted: true` entry from `pull` is how
> the client learns to delete locally.

### `POST /sync/push`

```ts
const PushRequest = z.object({
  syncVersion: z.number().int(),
  changes: z.array(z.object({
    opId:       z.string().uuid(),
    docId:      z.string().min(1).max(128),
    docType:    z.enum(["fact", "document", "verse", "setlist"]),
    wrappedDek: SealedPayload,
    ciphertext: z.string().regex(/^[A-Za-z0-9_-]+$/),
    iv:         z.string().regex(/^[A-Za-z0-9_-]{16,24}$/),
    algo:       z.literal("AES-256-GCM"),
    kdfVersion: z.number().int(),
    byteLength: z.number().int().min(0).max(10 * 1024 * 1024),
    baseRev:    z.number().int().min(0),
  })).min(1).max(500),
})
```

```jsonc
// 200
{
  "cursor": 418,
  "applied": [
    { "docId": "email",   "revision": 413, "status": "applied" },
    { "docId": "phone",   "revision": 414, "status": "applied" },
    { "docId": "address", "revision": 415, "status": "conflict" },
    { "docId": "pan",     "revision": 411, "status": "duplicate" }
  ]
}
```

**Four statuses, and the client must handle all four:**

| Status | Meaning | Client action |
|---|---|---|
| `applied` | Written, revision assigned | Bump local `baseRev` |
| `duplicate` | `opId` already seen — a retry | Delete from queue. **No error.** |
| `conflict` | Another device wrote it first | Client-side three-way merge |
| — (missing from `applied`) | Tombstoned since you last synced | Delete locally |

| Status | When |
|---|---|
| 200 | At least processed |
| 401 | Expired token |
| 409 `sync_version_mismatch` | `syncVersion` differs. `details.requiredSyncVersion` |
| 413 | A single blob over 10MB |
| 429 | Over 120/min |

> **`duplicate` is a success, not an error.** A phone on a train uploads, the server applies
> it, the response dies with a dead cell tower, and the client retries. Without `duplicate`, that
> retry either 500s or double-writes. **Treat `duplicate` as "already done, carry on."**

### `POST /sync/change-passphrase`

```ts
const ChangePassphraseRequest = z.object({
  wrappedDeks: z.array(z.object({
    docId: z.string().min(1).max(128),
    wrappedDek: SealedPayload,
  })).min(1).max(2000),
})
```

```jsonc
// 200
{ "rewrapped": 47, "cursor": 465 }
```

**This endpoint uploads wrapped keys only — never document ciphertext.** Changing your
passphrase on a 12MB document does not re-upload 12MB, because the DEK is random and the
ciphertext is untouched. Chapter 6 Step 7.

> **`max(2000)` and what happens past it.** A user with 3,000 documents cannot change their
> passphrase. That is a real limit and you should handle it honestly: return 422 with
> `{ "details": { "maxBatch": 2000, "total": 3140 } }` and let the client page through. Do not
> silently truncate — the user would believe their passphrase changed and half their documents
> would be unopenable. **Silent truncation on a key rotation is data loss.**

---

## Step 7 — Account

### `GET /account`

```jsonc
// 200
{
  "user": { "id": "...", "email": "...", "plan": "free", "createdAt": 1756012345678 },
  "storage": { "blobCount": 214, "byteLength": 48213904, "quotaBytes": 1073741824 },
  "devices": { "active": 2, "revoked": 1 },
  "sync": { "cursor": 465, "lastSyncAt": 1757012345678 }
}
```

**`byteLength` is metadata the server legitimately holds.** It powers the quota bar in Settings.
It reveals how much data you have, never what is in it — Chapter 5's boundary, correctly
crossed.

### `POST /account/export`

```ts
// 202 — it is a job, not a response
{ "requested": true, "expiresInSeconds": 300 }
```

```ts
account.post("/export", requireAuth, requireActiveSubscription, async (c) => {
  // NEVER return a presigned URL to your own Atlas credentials
  // for the blobs collection. That URL would decrypt nothing —
  // the server cannot decrypt either — but it WOULD hand out
  // read access to every ciphertext in the account. The export is
  // built from what the client already has: it pulls its own
  // blobs and writes a local archive.
  throw new AppError("not_found", "Export happens on your device.", 404)
})
```

> **The server cannot export your data, and pretending otherwise is the trap.** A tempting
> design is a "Download my data" button that streams every blob — but streaming ciphertext is
> useless to the user, and the streaming endpoint itself becomes a credential oracle.
>
> **The honest design: export is client-side.** The web app pulls its blobs, decrypts them, and
> writes `refrain-export-YYYY-MM-DD.json` to your Downloads folder. Chapter 16 Step 4 already
> implements this. The endpoint above exists only to return a clear 404 explaining why, so a
> confused user is not left wondering.

### `DELETE /account`

```ts
const DeleteAccountRequest = z.object({
  authKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/),
  /** Typed confirmation. Not a checkbox — a literal string. */
  confirm: z.literal("delete my account"),
})
```

```jsonc
// 202 — soft delete, hard delete after the grace period
{ "deletedAt": 1757012345678, "hardDeleteAt": 1757612345678 }
```

| Status | When |
|---|---|
| 202 | Soft-deleted. Revoked every device and token immediately. |
| 401 | `authKey` did not verify |
| 422 | `confirm` string did not match |

> **Deletion is immediate for access, delayed for data.** The 7-day grace period exists so a
> panicking user who clicks the wrong button can recover. But **every token and device is
> revoked right now**, not in seven days — otherwise "delete my account" and "I still have
> access" are both true, which is the worst possible state.
>
> `purgeUser` (Chapter 5 Step 7) runs at `hardDeleteAt` via the platform's scheduled task.

---

## Step 8 — Rate limits

| Group | Limit | Window | Why that number |
|---|---|---|---|
| `POST /auth/register` | 5 | 1 min | Brute force **and** enumeration |
| `POST /auth/login` | 5 | 1 min | Brute force and enumeration |
| `POST /auth/refresh` | 30 | 1 min | Chatty by design; not a credential guess |
| `PUT /auth/auth-key` | 5 | 1 min | Credential-changing is rare and dangerous |
| `GET /sync/pull` | 120 | 1 min | A client checks every 30s = 2/min |
| `POST /sync/push` | 120 | 1 min | Backlog bursts after a week offline |
| `POST /sync/blobs` | 120 | 1 min | Paired with pull |
| `GET /devices`, `/account` | 60 | 1 min | Normal browsing |
| `DELETE /account` | 3 | 1 hour | Irreversible |

```http
HTTP/1.1 429 Too Many Requests
Retry-After: 60
X-RateLimit-Limit: 5
X-RateLimit-Remaining: 0
X-RateLimit-Reset: 1757012400
```

```ts
// The client MUST honour Retry-After, and MUST NOT retry sooner.
const retryAfter = Number(res.headers.get("Retry-After") ?? "60")
const delay = Math.max(retryAfter, 2 ** attempt * 1000) + Math.random() * 250
```

> **`Retry-After` is mandatory on a 429 and clients ignore it constantly.** Without it, every
> client guesses — usually "immediately" — and your rate limiter turns into a self-inflicted
> denial of service. The `+ Math.random()` matters too: without jitter, every client that hits
> the limit at the same moment retries at the same moment, forever, in a synchronised wave.
>
> **`DELETE /account` at 3/hour is not arbitrary caution.** It is the one endpoint where a
> wrong click is unrecoverable. The limit exists so a client bug cannot delete an account three
> times and a retry storm cannot delete forty.

---

## Step 9 — The client

One `SyncSession` from Chapter 6, plus the error narrowing that makes the API typed.

```ts
// packages/vault/src/sync/api.ts
import { ApiError, type ApiError as ApiErrorT } from "@refrain/fields"

export class ApiClient {
  constructor(private session: SyncSession, private base = API_URL) {}

  async request<T>(
    path: string,
    schema: { safeParse(v: unknown): { success: true; data: T } | { success: false } },
    init: RequestInit = {},
  ): Promise<T> {
    const res = await this.session.fetch(`${this.base}${path}`, {
      ...init,
      headers: { "content-type": "application/json", "x-request-id": newRequestId(), ...init.headers },
    })

    if (!res.ok) throw await this.toApiError(res)
    return schema.parse(await res.json())      // validate responses too
  }

  private async toApiError(res: Response): Promise<ApiErrorT> {
    // A proxy or a platform router returns HTML, not your error
    // envelope. Parsing must not be the thing that throws.
    const parsed = ApiError.safeParse(await res.json().catch(() => null))
    if (!parsed.success) {
      return { error: { code: "internal", message: `Unexpected ${res.status} from the server.` } }
    }
    return parsed.data
  }
}

export const api = {
  pull: (since: number, docType?: string) =>
    api.request(`/sync/pull?since=${since}${docType ? `&docType=${docType}` : ""}`, PullResponse),

  push: (changes: PushChange[]) =>
    api.request("/sync/push", PushResponse, {
      method: "POST",
      body: JSON.stringify({ syncVersion: SYNC_VERSION, changes }),
    }),

  fetchBlobs: (docIds: string[]) =>
    api.request("/sync/blobs", BlobsResponse, {
      method: "POST", body: JSON.stringify({ docIds }),
    }),
}
```

### Validate responses, not just requests

```ts
// ❌ Trusting the server shape
const { entries } = await res.json()
entries.forEach(...)          // crashes at 3am when a field is renamed

// ✅ One schema, both directions
const PullResponse = z.object({
  entries: z.array(SyncEntrySchema),
  cursor: z.number().int(),
  hasMore: z.boolean(),
  serverTime: z.number().int(),
})
```

> **Validating responses is what makes a rolling deploy safe.** You deploy the server first with
> an added field; old clients parsing strictly would break. But strict parsing on *responses*
> plus a versioned *request* contract means you find out in your own test, not from a user's
> bug report. **Zod on both sides is why `@refrain/fields` exists.**

---

## Step 10 — Curl walkthrough

The whole flow, by hand. Run it once — you will understand the protocol better than from any
description.

```bash
# ── Health ──
curl -s localhost:8787/health
curl -s localhost:8787/health/ready

# ── Register ──
curl -s -X POST localhost:8787/auth/register \
  -H 'content-type: application/json' \
  -H 'x-device-label: Terminal' \
  -d '{"email":"dev@refrain.local","authKey":"'"$(openssl rand -base64 32 | tr '+/' '-_' | tr -d '=')"'"}'

export ACCESS="eyJ..."   # from the response
export REFRESH="8Jm2..."

# ── Who am I ──
curl -s localhost:8787/auth/me -H "authorization: Bearer $ACCESS"

# ── Push one fact ──
curl -s -X POST localhost:8787/sync/push \
  -H "authorization: Bearer $ACCESS" -H 'content-type: application/json' \
  -d '{
    "syncVersion": 1,
    "changes": [{
      "opId": "'"$(uuidgen | tr 'A-Z' 'a-z')"'",
      "docId": "email", "docType": "fact",
      "wrappedDek": {"ciphertext":"AQIDBAUGBwgJCgsMDQ4PEBESExQ=","iv":"AAAAAAAAAAAAAAAA"},
      "ciphertext": "3q2+7w8EAA",
      "iv": "AAAAAAAAAAAAAAAA",
      "algo": "AES-256-GCM", "kdfVersion": 1,
      "byteLength": 8, "baseRev": 0
    }]
  }'

# ── Retry the SAME push. Must report "duplicate", not write twice. ──
# (reuse the exact same body)

# ── Pull ──
curl -s "localhost:8787/sync/pull?since=0" -H "authorization: Bearer $ACCESS"

# ── Devices ──
curl -s localhost:8787/devices -H "authorization: Bearer $ACCESS"

# ── Refresh. The OLD refresh token is dead after this. ──
curl -s -X POST localhost:8787/auth/refresh \
  -H 'content-type: application/json' -d '{"refreshToken":"'"$REFRESH"'"}'
```

```bash
# ── The two audits that must always come back clean ──

# No ciphertext or personal content leaking into a metadata response.
curl -s "localhost:8787/sync/pull?since=0" -H "authorization: Bearer $ACCESS" \
  | grep -iE '"(ciphertext|value|label|filename|email|wrappedDek)"' \
  && echo "✗ LEAK in pull metadata" || echo "✓ pull returns metadata only"

# No field labels anywhere in the API logs.
grep -riE '"(label|value|pan|aadhaar|email)"\s*:' logs/api.log \
  && echo "✗ content in logs" || echo "✓ logs carry shape, not content"
```

> **Run both audits after every change to the sync routes.** They are two `grep`s and they catch
> the exact class of bug that destroys this product's central claim — the server being able to
> read what it is not supposed to read.

---

## Step 11 — Contract tests

The API is only real if a test fails when it drifts from this document.

```ts
// apps/api/src/routes/contract.test.ts
describe("API contract", () => {
  it("every route returns the error envelope, never a bare string", async () => {
    for (const [path, method] of ALL_ROUTES) {
      const res = await call(method, path, {}, { token: VALID })
      if (res.status >= 400) {
        expect(ApiError.safeParse(await res.json()).success).toBe(true)
      }
    }
  })

  it("every documented status code is reachable", async () => {
    // If a code in this chapter cannot be triggered, it is
    // documented fiction and will not work when you need it.
    expect(await canTrigger("sync_version_mismatch", 409)).toBe(true)
    expect(await canTrigger("payload_too_large", 413)).toBe(true)
    expect(await canTrigger("rate_limited", 429)).toBe(true)
  })

  it("every authenticated route rejects a missing token with 401", async () => {
    for (const [path, method] of PROTECTED_ROUTES) {
      const res = await call(method, path)
      expect(res.status).toBe(401)
      expect(res.headers.get("www-authenticate")).toBeTruthy()
    }
  })

  it("sync/pull never returns ciphertext", async () => {
    const res = await call("GET", "/sync/pull?since=0", {}, { token: VALID })
    const body = await res.text()
    expect(body).not.toMatch(/"ciphertext"/)
    expect(body).not.toMatch(/"wrappedDek"/)
    expect(body).not.toMatch(/"value"/)
  })

  it("a rate-limited response carries Retry-After", async () => {
    await exhaust("POST", "/auth/login")
    const res = await call("POST", "/auth/login", {})
    expect(res.status).toBe(429)
    expect(res.headers.get("retry-after")).toBeTruthy()
  })

  it("self-revocation is 403, not 422", async () => {
    // The 401-loop bug from Step 2, as a test.
    const me = await call("GET", "/auth/me", {}, { token: VALID })
    const myId = (await me.json()).device.id
    expect((await call("DELETE", `/devices/${myId}`, {}, { token: VALID })).status).toBe(403)
  })
})
```

```ts
// And a test that keeps the docs honest:
it("every route in app.ts appears in the reference", async () => {
  const implemented = ALL_ROUTES.map(([path, method]) => `${method} ${path}`)
  const documented = REFERENCE_ROUTES   // parsed from 16-api-reference.md
  expect(documented).toEqual(expect.arrayContaining(implemented))
})
```

> **That last test is unusual and it is worth writing.** A reference document drifts from the
> code the moment someone adds an endpoint and forgets to document it — and then it is worse
> than nothing, because people trust it. **Parsing your own markdown and asserting the routes
> match makes this chapter impossible to let rot.** It is 15 lines and it has caught real
> undocumented endpoints every time it is used.

---

## Step 12 — Commit

```bash
git add -A
git commit -m "feat(api): 12 endpoints, shared error schema, rate limits, contract tests"
```

Add two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | No `/v1/` path prefix; `SYNC_VERSION` in the payload instead | Sync is the only protocol with a real compatibility problem, and it solves it itself. A version path you never exercise is dead code. |
| 2026-10-XX | Revoking a device returns 403, not 401 | 401 tells the client to refresh, which also fails, which refreshes again — an infinite logout loop. |
| 2026-10-XX | `POST /account/export` returns 404; export is client-side | The server can only stream ciphertext, which is useless to the user, and the endpoint becomes a credential oracle. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The error catalogue and each code's HTTP status** | **You.** Clients branch on this |
| **The 401-vs-403 decision and the logout-loop reasoning** | **You** |
| **`duplicate` as a success, not an error** | **You** |
| **The `change-passphrase` 2000-doc limit and why silent truncation is data loss** | **You** |
| **The rate-limit table and each number's reasoning** | **You** |
| Why `pull` is metadata-only, in one sentence | **You** |
| Zod schemas for requests and responses | **OpenCode** from these templates |
| Route handler wiring | **OpenCode** |
| `ApiClient` and `toApiError` | **OpenCode** |
| The contract test suite | **OpenCode** — then **add one test it does not have** |
| The markdown-parsing documentation test | **OpenCode** |
| Curl walkthrough | **You**, by hand. This is how it becomes real. |

---

## Gotchas in this chapter

**Every authenticated route works without a token.** You forgot `requireAuth` on one route —
and it is always the one you tested least. The "rejects a missing token" contract test is what
catches it, and it catches it in one second instead of one incident.

**A 500 leaks a stack trace.** Your `onError` returns `err.message` for a non-`AppError`. Log
it, return a sentence. Check with `curl`, never with your own eyes.

**`pull` returns ciphertext.** You inlined the blob into the syncmeta response "to save a round
trip." You have just given every device a reason to download every document on every sync, and
destroyed the reason for the collection split.

**Revocation returns 401 and users get logged out in a loop.** See Step 2. This one is very
hard to spot in manual testing and obvious in the contract test.

**The client retries immediately on a 429.** No `Retry-After` header. You now have a
self-inflicted DoS. See Step 8.

**`Retry-After` is missing on 429.** Your rate-limit middleware threw an `AppError` instead of
setting the header. Verify with `curl -i`.

**A retry double-writes.** `opId` is being generated at *send* time instead of *queue* time, so
the retry looks like a new operation.

**Response shapes are not validated.** The server renamed a field, the client reads `undefined`,
and `entries.forEach` throws inside a `catch` that swallows it. Sync silently stops forever.

**The passphrase-change limit truncates silently.** 3,000 documents, `max(2000)`, no error. Half
the user's vault is now wrapped under a key they cannot derive. **This is the worst bug this
API can have.**

**`/account/export` streams ciphertext.** It "works", it returns 200, and it is useless — and it
is a read credential handed out for the whole account. See Step 7.

**Two `<nav>`s in the curl examples and none in the docs.** Just kidding. But `curl` without
`-i` hides status codes, and you will spend an hour wondering why a route seems to do nothing.
**Always `-i` when debugging.**

**`Content-Type: text/plain` from `curl -d`.** `curl -d` without `-H 'content-type:
application/json'` sends form encoding, your Zod validator rejects it, and you get a 422 about
a field that looks perfectly correct.

---

## Verify before moving on

- [ ] Every route in this document exists, and every implemented route is documented
- [ ] Every 4xx/5xx response is the error envelope — no bare strings, no stacks
- [ ] Every authenticated route returns 401 without a token, with `WWW-Authenticate`
- [ ] Self-revocation returns 403 and the client does not loop
- [ ] `sync/pull` contains no `ciphertext`, `wrappedDek`, `value`, or `label`
- [ ] A repeated `push` with the same `opId` reports `duplicate` and writes once
- [ ] `sync_version_mismatch` returns 409 with `details.requiredSyncVersion`
- [ ] Every documented status code is reachable in a test
- [ ] Every 429 carries `Retry-After`
- [ ] The client honours `Retry-After` and adds jitter
- [ ] `change-passphrase` rejects over 2000 documents with a **clear error**, never a truncation
- [ ] `change-passphrase` sends no document ciphertext
- [ ] `/account/export` returns 404 with an explanation
- [ ] `DELETE /account` revokes every token immediately, hard-deletes after 7 days
- [ ] Responses are Zod-validated on the client
- [ ] The doc-drift test fails when you add an undocumented route (verify by adding one)

---

## Check yourself before Chapter 8

1. **Why does `pull` return metadata and `blobs` return ciphertext, as two endpoints?**
2. **What does `status: "duplicate"` mean, and why is it not an error?**
3. **Why must revocation be 403 and not 401?**
4. **Why is `/auth/login` rate-limited at 5/min but `/sync/pull` at 120/min?**
5. **Why does `Retry-After` need to exist *and* client jitter?**
6. **What breaks if the passphrase-change batch limit truncates silently?**
7. **Why does `/account/export` return 404 instead of streaming?**
8. **Why validate responses with Zod when the server is yours?**
9. **What is the difference between liveness and readiness, and what does checking MongoDB in liveness cause?**
10. **Why does the token payload carry `sub` and `did` but never `email`?**

---

**Next: [Chapter 8 — CI/CD & Deployment](./18-cicd-deploy.md)** — GitHub Actions, the
extension zip versus the web deploy versus the API deploy, secrets, migrations, and what to do
when a deploy goes wrong at 1am.