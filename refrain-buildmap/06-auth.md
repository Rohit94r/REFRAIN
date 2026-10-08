# Chapter 6 — Auth

> **Day 6 · Goal: register, log in, rotate tokens, revoke one device.**
>
> The passphrase is never sent. The credential the server stores cannot derive the key that
> decrypts your data. Every other auth implementation detail follows from those two facts.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **Auth / authentication** — proving who someone is.
- **Passphrase** — your master secret. Used to derive keys. Never stored, never
  sent.
- **KDF (key derivation function)** — turns a short passphrase into a long,
  strong key. Deliberately slow so guessing is expensive.
- **PBKDF2** — the KDF your browser can use.
- **Argon2id** — a stronger, more modern KDF. Used on the server.
- **HKDF** — splits one key into several, each for a different job.
- **Access token** — a short-lived ticket (15 minutes). Sent with each request.
- **Refresh token** — a longer-lived ticket used to get new access tokens.
- **Token rotation** — every refresh token is single-use and replaced by a new
  one. If an old one is reused, someone stole it.
- **JWT** — a signed token that carries claims.
- **Asymmetric keys** — a public key that anyone can read, and a private key only
  you hold. Used to sign and verify.
- **Revocation** — making a token invalid before it expires.
- **401 vs 403** — 401 means "I do not know who you are"; 403 means "I know, and
  no".

---

## Understand this first

### The server authenticates a key it cannot invert

Chapter 5's diagram is the whole design. Restating it as a login sequence:

```
REGISTER
  client                                     server
    │  PBKDF2(passphrase, HMAC(email))            │
    ├──────────► k_root                          │
    │  HKDF(k_root, "auth")                       │
    ├──────────► k_auth ─────────────────────────►│
    │  HKDF(k_root, "vault")                      │  Argon2id(k_auth)
    │  k_vault  ✋ STAYS HERE                      ├──────────► authHash
    │                                              │

LOGIN
  client                                     server
    │  PBKDF2(passphrase, HMAC(email))            │
    ├──────────► k_root                          │
    │  HKDF(k_root, "auth") → k_auth               │
    ├──────────► k_auth ─────────────────────────►│  argon2.verify(k_auth, authHash)
    │  k_vault stays local, unwraps your DEKs     │  ✔ issue access + refresh
    │                                              │
```

**Three consequences you should be able to state without looking:**

**The server never receives the passphrase.** Not hashed, not salted, not derivable. There is no
`password` field in the `users` collection, and there is no endpoint that accepts one.

**A full database compromise does not yield decryptable data.** An attacker with
`refrain_prod` has `authHash` (Argon2id output) and `blobs.ciphertext`. To decrypt they need
`k_vault`, which requires the passphrase. Argon2id is expensive; PBKDF2 at 600k on top of it is
600k *more* expensive. That is the layered cost you are buying.

**The server cannot reset your passphrase.** There is no recovery flow, because recovering means
either holding your key or trusting someone who does. The recovery path is the recovery phrase
in Chapter 12, generated client-side, never uploaded.

> **If you ever add a "forgot password" email, you have broken the entire architecture.** A
> server-side reset requires the server to hold something that can re-derive `k_root`. Design
> the flow so that account deletion is the only server-side recovery, and everything else is
> client-side. Write this into §11 as an explicit consequence, because a well-meaning
> contributor will otherwise try to be helpful and destroy the product.

### Asymmetric JWTs, because the extension must verify offline

| | HS256 (symmetric) | EdDSA (asymmetric) |
|---|---|---|
| Verifier needs | The signing secret | Only the public key |
| Server compromise | Attacker can mint any token | Attacker can read, not mint |
| Client can verify | ❌ Needs a secret | ✅ Public key is public |
| Key size | 256-bit shared | Ed25519: 32-byte private, 32-byte public |

**Why the client verifying matters here.** A content script runs on pages you do not control. If
the side panel cannot check a token's validity locally, every panel action is a round trip to a
server it is trying to trust less. With EdDSA, the extension ships the public key, verifies in
microseconds, and needs the network only for actual data.

And the security property is strictly better: a stolen API token grants *your* access, not the
attacker's ability to become you.

---

## Step 1 — Keys

```bash
mkdir -p apps/api/keys && cd apps/api/keys
openssl genpkey -algorithm ed25519 -out ed25519-private.pem
openssl pkey -in ed25519-private.pem -pubout -out ed25519-public.pem
```

> **The private key lives on the host, injected as an env var, never in git, never in the
> bundle.** A 32-byte Ed25519 key is small enough that you can store it in a secrets manager
> without thinking about it. The public key is served from `/auth/jwks.json` and is genuinely
> public — publish it, cache it hard.

```ts
// apps/api/src/lib/keys.ts
import { createPrivateKey, createPublicKey, SignJWT, jwtVerify, type JWTPayload } from "jose"
import { readFileSync } from "node:fs"
import { env } from "../env"

const privPem = env.JWT_PRIVATE_KEY_PEM.replace(/\\n/g, "\n")
const priv = await importPKCS8(privPem, "EdDSA")
const pub = await importSPKI(env.JWT_PUBLIC_KEY_PEM.replace(/\\n/g, "\n"), "EdDSA")

export const ACCESS_TTL_SECONDS = 15 * 60        // 15 minutes
export const REFRESH_TTL_DAYS = 30

export interface AccessClaims extends JWTPayload {
  sub: string       // userId
  did: string       // deviceId
}

export async function signAccessToken(claims: AccessClaims): Promise<string> {
  return new SignAccessJWT(claims)
    .setProtectedHeader({ alg: "EdDSA", kid: KEY_ID })
    .setIssuedAt()
    .setExpirationTime(`${ACCESS_TTL_SECONDS}s`)
    .setIssuer("https://api.refrain.dev")
    .setAudience("refrain")
    .sign(priv)
}

export async function verifyAccessToken(token: string): Promise<AccessClaims> {
  const { payload } = await jwtVerify(token, pub, {
    issuer: "https://api.refrain.dev",
    audience: "refrain",
    algorithms: ["EdDSA"],       // ← pin. Never trust the header's alg.
  })
  return payload as AccessClaims
}
```

> **`algorithms: ["EdDSA"]` is a hard requirement, not a default.** The classic JWT attack sends
> `{"alg": "none"}` or `{"alg":"HS256"}` with the public key in the `kid` header, and a naive
> verifier signs a forged token with the public key it is holding. **Pinning the expected
> algorithm** closes it. Every hand-rolled JWT verifier in the history of the internet has this
> bug.

> **`kid: KEY_ID` from the start.** Add key rotation support now. Rotating a signing key is an
> eventuality, not a hypothetical, and `kid` is what makes it a config change instead of an
> outage. Do this before you need it.

### Never put the email in the JWT

```ts
// ❌ PII, base64-decodable by anyone who sees the token, and
//    access tokens get logged by proxies and CDNs.
{ sub, email: "rohit@example.com", iat, exp }

// ✅
{ sub: userId, did: deviceId, iat, exp, jti }
```

A JWT is not encrypted. Anything you put in it is readable by your infrastructure, by anyone
who intercepts it, and by anyone with access to a log line. `sub` is an ObjectId — useless
without database access.

---

## Step 2 — Argon2id

```ts
// apps/api/src/lib/hash.ts
import * as argon2 from "argon2"

// OWASP's recommended Argon2id configuration.
const OPTS = {
  type: argon2.argon2id,
  memoryCost: 19_456,     // 19 MiB
  timeCost: 2,
  parallelism: 1,
} as const

/** Lower cost for tests only. NEVER branch on NODE_ENV in production code. */
const TEST_OPTS = { ...OPTS, memoryCost: 512, timeCost: 1 } as const

export const hashAuth = (key: Uint8Array) =>
  argon2.hash(Buffer.from(key), env.NODE_ENV === "test" ? TEST_OPTS : OPTS)

export const verifyAuth = (key: Uint8Array, hash: string) =>
  argon2.verify(Buffer.from(key), hash)

/** Refresh tokens are high-entropy already. No KDF needed — just SHA-256. */
export const hashRefreshToken = (token: string) =>
  createHash("sha256").update(token).digest("hex")
```

**Three decisions:**

**Argon2id, not bcrypt or scrypt.** Argon2id is memory-hard *and* time-hard, and it is the OWASP
first recommendation. bcrypt's 72-byte input limit silently truncates anything longer. scrypt
requires you to pick parameters and get them wrong.

**`memoryCost: 19_456` = 19 MiB.** That is the OWASP baseline. It is tuned so a single login is
indistinguishable from normal work on any device from the last decade.

**SHA-256 for refresh tokens, not Argon2.** Argon2 is for low-entropy human-chosen secrets.
A 256-bit random token does not need a slow hash — the entropy is already there, and running
Argon2 on every refresh on every request would cost you more than it buys. **Use the right
primitive for the input, not one primitive for everything.**

```ts
export function generateRefreshToken(): { token: string; hash: string } {
  const token = randomBytes(32).toString("base64url")   // 256 bits, URL-safe
  return { token, hash: hashRefreshToken(token) }
}
```

---

## Step 3 — Registration

```ts
// apps/api/src/routes/auth.ts
auth.post("/register", limitAuth, zValidator("json", RegisterSchema), async (c) => {
  const { email, authKey } = c.req.valid()

  const existing = await UserModel.findOne({ email, deletedAt: null }).lean()
  if (existing) {
    // Do not confirm or deny. Same message either way.
    throw new AppError("validation_failed", "Could not create that account.", 422)
  }

  const authHash = await hashAuth(base64ToBytes(authKey))

  const device = { label: c.req.header("x-device-label") ?? "Unknown device", platform: detectPlatform(c) }

  const user = await UserModel.create({ email, authHash })
  const dev = await DeviceModel.create({
    userId: user._id, ...device, fingerprint: fingerprintOf(c),
  })
  user.devices.push(dev._id)
  await user.save()

  const { token: refresh, hash } = generateRefreshToken()
  await RefreshTokenModel.create({
    userId: user._id, deviceId: dev._id, tokenHash: hash,
    expiresAt: addDays(new Date(), REFRESH_TTL_DAYS),
  })

  return c.json({
    accessToken: await signAccessToken({ sub: String(user._id), did: String(dev._id) }),
    refreshToken: refresh,
    user: publicUser(user),
  }, 201)
})
```

```ts
// packages/vault/src/sync/client.ts — the client side
export async function register(email: string, passphrase: string) {
  const k_root = await deriveRoot(passphrase, email)
  const k_auth = await exportRaw(await deriveAuthKey(k_root))

  const res = await fetch(`${API}/auth/register`, {
    method: "POST",
    headers: { "content-type": "application/json", "x-device-label": navigator.platform },
    body: JSON.stringify({ email, authKey: toBase64(k_auth) }),
  })
  // k_root and k_vault are now out of scope. They never left this function.
}
```

> **The comment at the bottom is load-bearing documentation.** Six months from now, someone will
> refactor `register()` and add `passphrase` to the body "for debugging." That comment is the
> thing that stops them. Write the *reason*, not just the code.

### Three things a registration endpoint gets wrong

**1. Confirming whether an email exists.**

```ts
// ❌ Leaks the entire user list to anyone with a script.
if (existing) return c.json({ error: "That email is already registered" }, 409)
```

`/register` is an account-enumeration oracle. The fix is one generic message and a
**consistent response time** — return 422 with the same shape whether or not the user exists.
Rate limiting (5/min) is the other half.

**2. Not being async-uniform.** If the "user does not exist" path returns in 3ms and the "user
exists, verifying hash" path takes 400ms, the timing leaks existence even with the same
message. Argon2id's cost is a security feature *and* a timing side-channel. Verify a dummy hash
on the not-found path.

**3. No `deletedAt: null` in the lookup.** If a user deleted their account, their email is
still in the collection with `deletedAt` set. A plain `findOne({ email })` says "already
registered" to someone trying to sign up fresh. The soft-delete filter is required here too.

---

## Step 4 — Login

```ts
auth.post("/login", limitAuth, zValidator("json", LoginSchema), async (c) => {
  const { email, authKey } = c.req.valid()
  const user = await UserModel.findOne({ email, deletedAt: null }).select("+authHash")

  // ── Constant-ish work on BOTH paths. See "not being async-uniform". ──
  const hash = user?.authHash ?? DUMMY_HASH
  const ok = await verifyAuth(base64ToBytes(authKey), hash)

  if (!user || !ok) {
    // Identical response for "no such user" and "wrong key".
    throw new AppError("unauthorized", "Email or passphrase is incorrect.", 401)
  }
  ...
})
```

```ts
// A real Argon2id hash, generated once at boot, of random bytes.
// Verifying against it burns the same CPU as a real verify, so
// the not-found path and the wrong-key path cost the same.
const DUMMY_HASH = await hashAuth(crypto.getRandomValues(new Uint8Array(32)))
```

> **This is the most under-implemented detail in most auth systems.** If the not-found path
> returns fast and the wrong-password path returns slow, an attacker enumerates your user list
> in an afternoon with no credentials at all. One dummy hash at boot costs nothing and closes it.

---

## Step 5 — Refresh rotation with reuse detection

```ts
auth.post("/refresh", rateLimitRefresh, zValidator("json", RefreshSchema), async (c) => {
  const { refreshToken } = c.req.valid()
  const hash = hashRefreshToken(refreshToken)

  const current = await RefreshTokenModel.findOne({ tokenHash: hash })

  if (!current || current.revokedAt) {
    // ── REUSE DETECTION ──
    // A revoked token was presented. Either the user has two
    // browsers fighting over one refresh token, or someone stole it.
    // Assume theft. Revoke the entire family for that device.
    if (current) {
      await revokeDeviceFamily(current.userId, current.deviceId)
      logger.warn({ userId: current.userId, deviceId: current.deviceId }, "refresh token reuse")
    }
    throw new AppError("unauthorized", "Session expired. Please sign in again.", 401)
  }

  if (current.expiresAt < new Date()) {
    throw new AppError("unauthorized", "Session expired. Please sign in again.", 401)
  }

  // ── ROTATE ──
  const next = generateRefreshToken()
  const created = await RefreshTokenModel.create({
    userId: current.userId,
    deviceId: current.deviceId,
    tokenHash: next.hash,
    expiresAt: addDays(new Date(), REFRESH_TTL_DAYS),
    rotatedTo: current._id,
  })
  current.revokedAt = new Date()
  await current.save()

  return c.json({
    accessToken: await signAccessToken({ sub: String(current.userId), did: String(current.deviceId) }),
    refreshToken: next.token,
  })
})
```

```ts
/** Revoke every token for a device, including live ones. */
export async function revokeDeviceFamily(userId: string, deviceId: string) {
  await RefreshTokenModel.updateMany(
    { userId, deviceId, revokedAt: null },
    { $set: { revokedAt: new Date() } },
  )
}
```

### Why rotation with reuse detection

**Without rotation:** a stolen refresh token is valid for 30 days, silently, forever, and the
legitimate user never notices. Rotation alone fixes that — each token is single-use.

**With rotation but no reuse detection:** two legitimate tabs refreshing simultaneously will
occasionally race, one will present an already-rotated token, and the system will treat it as
theft and log the user out. You will get a support ticket titled "Refrain logs me out every
few minutes" and you will not enjoy debugging it.

**Rotation + reuse detection is correct**, with one tuning knob: on reuse detection, revoke the
whole device family. Two racing tabs produce a token that *was* revoked, but they do not present
a *stolen* token — the user's device is the same device. Revoking the family logs out that
device only, and the next login recovers immediately. **Worse than ideal UX, dramatically better
than silent theft.**

> **Do not revoke the whole account on reuse.** It is the tempting "safe" choice and it punishes
> every user who has a race condition. Revoke the device. The user logs in again on one machine,
> not four.

---

## Step 6 — The middleware

```ts
// apps/api/src/middleware/auth.ts
export async function requireAuth(c: Context, next: Next) {
  const header = c.req.header("authorization")
  if (!header?.startsWith("Bearer ")) {
    throw new AppError("unauthorized", "Not signed in.", 401)
  }

  let claims: AccessClaims
  try {
    claims = await verifyAccessToken(header.slice(7))
  } catch {
    throw new AppError("unauthorized", "Session expired.", 401)
  }

  // A valid signature is not enough. The device may be revoked.
  const device = await DeviceModel.findOne({
    _id: claims.did, userId: claims.sub, revokedAt: null,
  }).lean()
  if (!device) {
    throw new AppError("unauthorized", "This device has been signed out.", 401)
  }

  c.set("userId", claims.sub)
  c.set("deviceId", claims.did)
  await next()
}

/** Every sync route uses this, not just requireAuth. */
export async function requireActiveSubscription(c: Context, next: Next) {
  const user = await UserModel.findById(c.get("userId")).lean()
  if (!user) throw new AppError("unauthorized", "Account not found.", 401)
  if (user.plan === "free" && c.get("deviceId")) {
    await assertDeviceWithinFreeLimit(user, c.get("deviceId"))
  }
  await next()
}
```

> **The device-revocation check on every request is the reason "sign out this laptop" works.**
> A stateless JWT stays valid until it expires — up to 15 minutes. Without this check, revoking
> a device you suspect was stolen buys you 15 minutes. With it, the check is one indexed query
> and revocation is instant.

---

## Step 7 — Passphrase change

The elegant part, and a direct payoff of the HKDF split.

```ts
// packages/vault/src/keys/rotate.ts

/**
 * Changing the passphrase re-wraps every DEK. It does NOT re-encrypt
 * a single document.
 *
 * Because: DEKs are random and independent of k_vault. Changing
 * k_vault only changes the wrapper around each DEK. Document
 * ciphertext is untouched — nothing is re-uploaded.
 */
export async function changePassphrase(
  oldPassphrase: string, newPassphrase: string, email: string,
  docs: Array<{ docId: string; wrappedDek: Sealed; ciphertext: ArrayBuffer; iv: Uint8Array }>,
) {
  const oldRoot = await deriveRoot(oldPassphrase, email)
  const newRoot = await deriveRoot(newPassphrase, email)
  const oldVault = await deriveVaultKey(oldRoot)
  const newVault = await deriveVaultKey(newRoot)

  const rewrapped: typeof docs = []

  for (const d of docs) {
    const dek = await unwrapDEK(d.wrappedDek, oldVault)
    rewrapped.push({ ...d, wrappedDek: await wrapDEK(dek, newVault) })
  }

  // Only the wrappedDek changes. The ciphertext is byte-identical.
  const newAuthKey = await exportRaw(await deriveAuthKey(newRoot))

  return {
    authKey: toBase64(newAuthKey),
    updates: rewrapped.map((d) => ({ docId: d.docId, wrappedDek: d.wrappedDek })),
  }
}
```

**What this gets you for free:**

| Change | Cost | Re-encrypt? | Re-upload? |
|---|---|---|---|
| Passphrase | Re-wrap N DEKs locally (~ms each) | ❌ No | Only the wrapped keys |
| Add a device | Download all + unwrap | ❌ No | ❌ No |
| Revoke a device | One DB write | ❌ No | ❌ No |
| Delete a document | Delete two rows | ❌ No | ❌ No |

**None of those require decrypting and re-encrypting document bodies.** That is the payoff of
per-document random DEKs, and it is why the key hierarchy in Chapter 5 is not over-engineering
— it is the difference between a passphrase change taking 200ms and taking an hour.

```ts
// The client flow
await changePassphrase(old, new, email, docs)
  → PUT /auth/auth-key       { authKey }        // new login credential
  → PATCH /sync/blobs        [{ docId, wrappedDek }] × N   // wrapped keys only
```

> **Order matters: update the blobs first, then the auth key.** If you update `authHash` first and
> the blob re-wrapping fails halfway, you have documents wrapped under a key you no longer have
> — which means permanent data loss for a subset of your vault. Fail the blob update and your
> auth key still matches your old passphrase, so you can retry safely.

---

## Step 8 — Revoking a device

```ts
auth.delete("/devices/:id", requireAuth, async (c) => {
  const deviceId = c.req.param("id")

  // Cannot revoke the device you are using. That is a logout.
  if (deviceId === c.get("deviceId")) {
    throw new AppError("validation_failed", "You cannot revoke this device from itself.", 422)
  }

  await DeviceModel.updateOne(
    { _id: deviceId, userId: c.get("userId") },
    { $set: { revokedAt: new Date() } },
  )
  await revokeDeviceFamily(c.get("userId")!, deviceId)

  // Note: the DATA is not deleted. That is a separate, explicit action.
  return c.json({ revoked: deviceId })
})
```

**Revoking a device is not deleting its data.** Revoking kills access. The synced documents stay
in your vault, and the next sync from any other device re-uploads whatever is missing.

This is a distinction users get wrong constantly, and getting it wrong means either losing data
or keeping access you meant to cut. Chapter 19's account page needs both buttons, clearly
labelled, with different consequences spelled out.

---

## Step 9 — The client loop

```ts
// packages/vault/src/sync/session.ts
export class SyncSession {
  #access: { token: string; expiresAt: number } | null = null
  #inflight: Promise<Response> | null = null

  async fetch(path: string, init: RequestInit = {}): Promise<Response> {
    // Expire 60s early. Clock skew and latency are real.
    if (!this.#access || Date.now() > this.#access.expiresAt - 60_000) {
      await this.#refresh()
    }
    return this.raw(path, {
      ...init,
      headers: {
        ...init.headers,
        authorization: `Bearer ${this.#access!.token}`,
      },
    })
  }

  /** Deduplicate concurrent refreshes. Two tabs must not both rotate. */
  async #refresh() {
    this.#inflight ??= this.doRefresh().finally(() => { this.#inflight = null })
    await this.#inflight
  }
}
```

> **`#inflight` is not an optimisation, it is the reuse-detection safety net.** Two components
> firing 401s at the same moment both call `/refresh`, both present the same token, one is
> flagged as reuse, and the server revokes the device family. The user gets logged out for a
> network blip. **Deduplicate the refresh, and you never see this bug in production** — which
> means you will never find it, so write the test that forces it.

```ts
it("does not log the user out on a simultaneous 401 from two components", async () => {
  // Two parallel requests, one access token, refresh must happen once.
  const [a, b] = await Promise.all([
    session.fetch("/sync/changes"),
    session.fetch("/sync/changes"),
  ])
  expect(a.status).toBe(200)
  expect(b.status).toBe(200)
  // The user is still signed in. Not 401. Not logged out.
})
```

---

## Step 10 — Commit

```bash
git add -A
git commit -m "feat(auth): ed25519 JWTs, argon2id, refresh rotation with reuse detection, device revocation"
```

Add three rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | The server stores Argon2id(k_auth) and never the passphrase | A DB compromise yields no decryptable data, and the server cannot reset your passphrase. There is no "forgot password" email, by design. |
| 2026-10-XX | Not-found and wrong-key paths do identical work | Otherwise `/login` is an account-enumeration oracle driven purely by response timing. |
| 2026-10-XX | Passphrase change re-wraps DEKs, never re-encrypts documents | Order: blobs first, then the auth key. Reversed, a partial failure loses data. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The key hierarchy as a login sequence, and its three consequences** | **You.** This is the security architecture |
| **`deriveRoot` / `deriveVaultKey` / `deriveAuthKey`** | **You.** Never paste this |
| **`changePassphrase` and the ordering decision** | **You.** Getting it wrong loses data |
| The `DUMMY_HASH` constant-time login | **You** |
| Argon2id parameters and why SHA-256 is right for refresh tokens | **You** |
| The reuse-detection logic and why you revoke a device, not an account | **You** |
| Zod schemas for the request bodies | **OpenCode** |
| Route handlers, boilerplate around your checks | **OpenCode** |
| The `#inflight` refresh dedupe | **OpenCode**, then write the parallel-401 test yourself |
| Device routes | **OpenCode** |

---

## Gotchas in this chapter

**`jwtMalformed` on every request.** The `.pem` has escaped newlines in the env var. Replace
`\\n` with real newlines when reading, or sign a key that is one long line.

**A forged token with `{"alg":"none"}` is accepted.** You are not pinning
`algorithms: ["EdDSA"]`. See Step 1.

**The extension cannot verify tokens.** You used HS256. Asymmetric keys are what let the client
verify offline.

**Every login is 400ms and CI times out.** Argon2id at 19MiB, correct, and it does not
parallelise in tests. Set the test cost factor lower — in test config only.

**The user gets logged out every few minutes.** Two components racing a refresh. `#inflight` is
missing. Or: two tabs sharing one refresh token, and reuse detection revokes the family. Fix the
dedupe first; if it persists, check for a shared localStorage token across tabs.

**`authHash` appears in a log line.** Your scrubber covers `password` and `secret` but the field
is named `authHash`. Add it to `NEVER_LOG`.

**Registration says "email already registered".** Enumeration oracle. Use the generic message —
and remember the timing channel, which is the one people forget.

**A user changed their passphrase and lost half their documents.** You updated `authHash` before
the blob re-wrapping. Order: blobs first. See Step 7.

**A revoked device keeps syncing.** Your `requireAuth` only verified the JWT signature and
skipped the device check. That check is the whole feature.

**`MongoServerSelectionError` inside the auth route.** You used a model before `connectDb()`
resolved in `app.ts`. Await the connection before mounting routes.

---

## Verify before moving on

- [ ] Registration never receives or stores the passphrase — verified with a network inspector
- [ ] `users` has no `password` field, and no endpoint accepts one
- [ ] A wrong passphrase cannot decrypt anything
- [ ] `/login` responds identically for unknown-email and wrong-key, **including timing**
- [ ] Access tokens expire in 15 minutes
- [ ] Refresh tokens are single-use; reuse revokes the device family
- [ ] Reuse detection does **not** revoke other devices
- [ ] Revoking a device kills its access immediately, not after 15 minutes
- [ ] Revoking a device does not delete its data
- [ ] `changePassphrase` re-wraps without re-encrypting — assert ciphertext is byte-identical
- [ ] Blobs update before the auth key on passphrase change
- [ ] Two parallel 401s do not log the user out
- [ ] `verifyAccessToken` rejects `alg: none`
- [ ] Nothing PII is in a JWT payload
- [ ] `authHash` never appears in a log

---

## Check yourself before Chapter 7

1. **What does a full database compromise give an attacker, and what does it not?**
2. **Why can there be no "forgot passphrase" email?**
3. **Why does `DUMMY_HASH` exist, and what attack does it stop?**
4. **Why rotate refresh tokens at all if you have 15-minute access tokens?**
5. **Why revoke a device rather than the whole account on reuse detection?**
6. **What does passphrase change re-encrypt, and what does it skip?**
7. **Why must blob re-wrapping happen before the auth key update?**
8. **Why does `requireAuth` query the database on every request if the JWT is stateless?**
9. **Why is `SHA-256` right for refresh tokens but wrong for `k_auth`?**
10. **What is the difference between revoking a device and deleting its data?**

---

**Next: [Chapter 7 — The Sync Engine](./08-sync-engine.md)** — the cursor, push and pull, the
three-way merge that does not lose edits, and the offline-first client queue.