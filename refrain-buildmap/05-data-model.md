# Chapter 5 — The Data Model

> **Day 5 · Goal: every Mongoose schema written, indexed, and reasoned about.**
>
> Six collections. Five contain no personal content. That ratio is the design, and it is the
> thing a security reviewer will check first.

---

## Understand this first

### Hybrid storage — and why one collection cannot work

Chapter 4's conclusion: **a server that cannot read your data cannot query your data.** You
cannot index ciphertext. So you split the problem in two.

| Collection | Holds | Server can read it? | Size | Query pattern |
|---|---|---|---|---|
| `blobs` | Encrypted payloads + wrapped DEK | ❌ No | Large | Point reads by `(userId, docId)` |
| `syncmeta` | Revisions, timestamps, tombstones | ✅ Yes | Small, hot | Range scans by revision |

**Why two collections and not one document with both parts.** Because they have opposite access
patterns. `syncmeta` is queried on *every* sync — a small range scan over recent revisions. If
it lived inside the same documents as the ciphertext, every sync would drag megabytes of opaque
bytes through the working set to read a 40-byte revision number. `syncmeta` stays resident in
MongoDB's cache; `blobs` lives on disk and is fetched only when a document actually changed.

**The contract that makes this safe:** `syncmeta` contains no personal content. Ever. No field
names, no labels, no values, no filenames. It is routing information and nothing else.

> **Enforce it with a schema.** Make `syncmeta`'s Mongoose schema `strict: true` with an
> explicit field list and no catch-all, so a well-meaning contributor cannot add
> `title: String` six months from now. Security properties that depend on discipline decay;
> properties enforced by the schema do not.

### The key hierarchy — four keys, two of which never leave

This is the crypto model, and it is worth getting exactly right because everything else depends
on it.

```
passphrase  (typed by the human, never stored anywhere, never transmitted)
   │
   │  PBKDF2-SHA256(passphrase, salt = HMAC(email), 600,000 iters)
   ▼
k_root  ── never transmitted ─────────────────────────────────┐
   │                                                         │
   │  HKDF-SHA256(k_root, info = "refrain:vault:v1")        │
   ▼                                                         ▼
k_vault                                            k_auth  ── transmitted over TLS
   │                                                  │      (the server stores
   │  wraps each document's DEK                        │       Argon2id of it)
   ▼                                                  ▼
DEK  ── AES-256-GCM, per document, random             login credential
   │
   │  encrypts
   ▼
ciphertext  ── uploaded, opaque, forever
```

**Four properties this buys, and each one is load-bearing:**

**1. The server can authenticate you without ever being able to decrypt you.** `k_auth` is sent
as a credential; `k_vault` never is. The server has `Argon2id(k_auth)`. Even a full database
compromise plus full application compromise yields `k_auth`, which derives nothing.

**2. Changing your password does not mean re-encrypting your vault.** The server credential is
derived from `k_root`, and rotating it is a write to `users`. Your vault bytes are untouched.
This is why the architecture keeps the two separate rather than using the passphrase directly as
the password.

**3. Per-document DEKs enable selective sync.** You probably do not want your Aadhaar scan
leaving the device. If every document were encrypted directly with `k_vault`, the choice was
all-or-nothing. With a wrapped DEK per document, selective sync is just "do not upload that
one" — and no key management changes.

**4. `k_auth` is not a password a human chose.** It is PBKDF2 output, so a weak passphrase still
produces a high-entropy credential. Argon2id on top means brute force needs both.

```ts
// packages/vault/src/keys/derive.ts — THIS RUNS ONLY ON THE CLIENT
const PBKDF2_ITERS = 600_000

export async function deriveRoot(
  passphrase: string, email: string,
): Promise<CryptoKey> {
  // Salt = HMAC(email). Deterministic, so a returning user derives the
  // same k_root without the server ever storing a salt.
  const emailKey = await crypto.subtle.importKey(
    "raw", new TextEncoder().encode(email), { name: "HMAC", hash: "SHA-256" },
    false, ["sign"],
  )
  const salt = new Uint8Array(await crypto.subtle.sign("HMAC", emailKey, new TextEncoder().encode("refrain-kdf-v1")))

  const base = await crypto.subtle.importKey(
    "raw", new TextEncoder().encode(passphrase), "PBKDF2", false, ["deriveBits"],
  )
  const bits = await crypto.subtle.deriveBits(
    { name: "PBKDF2", hash: "SHA-256", salt: salt as BufferSource, iterations: PBKDF2_ITERS },
    base, 256,
  )
  return crypto.subtle.importKey("raw", bits, "HKDF", false, ["deriveKey"])
}

async function hkdf(root: CryptoKey, info: string, usage: KeyUsage[]): Promise<CryptoKey> {
  return crypto.subtle.deriveKey(
    { name: "HKDF", hash: "SHA-256", salt: new Uint8Array(0), info: new TextEncoder().encode(info) },
    root, { name: "AES-GCM", length: 256 }, false, usage,
  )
}

/** Encrypts documents. Never leaves the device. */
export const deriveVaultKey = (root: CryptoKey) =>
  hkdf(root, "refrain:vault:v1", ["encrypt", "decrypt"])

/** Sent to the server as the login credential. Derives nothing. */
export const deriveAuthKey = (root: CryptoKey) =>
  hkdf(root, "refrain:auth:v1", ["sign"])

// packages/vault/src/keys/dek.ts
/** A fresh key per document. Random means compromise is per-document. */
export const generateDEK = (): CryptoKey =>
  crypto.subtle.generateKey({ name: "AES-GCM", length: 256 }, true, ["encrypt", "decrypt"])

/** DEK encrypted under k_vault. Travels with the ciphertext. */
export async function wrapDEK(dek: CryptoKey, kVault: CryptoKey): Promise<{ ciphertext: ArrayBuffer; iv: Uint8Array }> {
  const raw = await crypto.subtle.exportKey("raw", dek)
  const iv = crypto.getRandomValues(new Uint8Array(12))
  const ciphertext = await crypto.subtle.encrypt({ name: "AES-GCM", iv: iv as BufferSource }, kVault, raw)
  return { ciphertext, iv }
}

export async function unwrapDEK(wrapped: { ciphertext: ArrayBuffer; iv: Uint8Array }, kVault: CryptoKey): Promise<CryptoKey> {
  const raw = await crypto.subtle.decrypt({ name: "AES-GCM", iv: wrapped.iv as BufferSource }, kVault, wrapped.ciphertext)
  return crypto.subtle.importKey("raw", raw, "AES-GCM", true, ["encrypt", "decrypt"])
}
```

> **`exportKey("raw", dek)` requires `extractable: true`, which contradicts Chapter 12's rule.**
> That is not a contradiction — it is the one legitimate exception, and it exists precisely so
> the DEK can be *wrapped*. Two different jobs, two different extractability. `k_root` and
> `k_vault` stay `false`; only the per-document DEK is `true`. If you ever find yourself setting
> `k_vault` to extractable, you have misunderstood which key does what.
>
> **Random DEKs mean per-document blast radius.** If one document's DEK is compromised — a
> leaked backup, a buggy export path — exactly one document is exposed. With a single shared key,
> one leak is everything.

---

## Step 1 — `users`

```ts
// apps/api/src/db/models/user.ts
import mongoose, { Schema, type InferSchemaType } from "mongoose"

const userSchema = new Schema({
  email:      { type: String, required: true, lowercase: true, trim: true },
  /** Argon2id of k_auth. Never the passphrase, never k_vault. */
  authHash:   { type: String, required: true, select: false },

  plan:       { type: String, enum: ["free", "supporter"], default: "free" },

  devices: [{ type: Schema.Types.ObjectId, ref: "Device" }],

  createdAt:  { type: Date, default: Date.now },
  lastSeenAt: { type: Date, default: Date.now },
  deletedAt:  { type: Date, default: null },
}, {
  timestamps: true,
  strict: true,
  // Belt and braces. Even if a query slips past sanitizeFilter.
  sanitizeFilter: true,
})

userSchema.index({ email: 1 }, { unique: true, partialFilterExpression: { deletedAt: null } })

export type User = InferSchemaType<typeof userSchema>
export const UserModel = mongoose.model("User", userSchema)
```

**Four decisions:**

**`lowercase: true, trim: true` on the schema, not in the handler.** Normalising at the schema
layer means every write path gets it, including seed scripts and future admin endpoints. Doing
it in the handler means you will forget it in one of them.

**`select: false` on `authHash`.** Any query without an explicit `.select("+authHash")` cannot
accidentally return the hash. If you add a `GET /users/:id` admin route, the credential is not
in the payload. This is how you avoid an accidental credential leak in a response body.

**`partialFilterExpression` on the unique index.** With a soft-delete `deletedAt`, a plain
unique index means a user can never re-register an address they deleted. A partial index frees
it while the account is live. **This is the standard fix and it is easy to forget**, because it
only breaks months later when someone tries to sign up again with their old email.

**`deletedAt` for soft delete.** §18's right-to-be-forgotten flow needs a grace period before
hard deletion. A hard delete cascades across six collections and cannot be undone, and you do
not want that power available from a single API call.

---

## Step 2 — `blobs`

```ts
// apps/api/src/db/models/blob.ts
const blobSchema = new Schema({
  userId:  { type: Schema.Types.ObjectId, ref: "User", required: true },
  /** Client-generated UUID. The server never mints document ids. */
  docId:   { type: String, required: true },
  docType: { type: String, enum: ["fact", "document", "verse", "setlist"], required: true },

  /* ── The opaque part. Nothing below this line is readable by us. ── */
  wrappedDek: {
    ciphertext: { type: Buffer, required: true },
    iv:          { type: Buffer, required: true },
  },
  ciphertext: { type: Buffer, required: true },
  iv:         { type: Buffer, required: true },
  algo:       { type: String, enum: ["AES-256-GCM"], default: "AES-256-GCM", required: true },
  kdfVersion: { type: Number, default: 1, required: true },

  /* ── Routing metadata. No personal content, ever. ── */
  byteLength: { type: Number, required: true, min: 0, max: 10 * 1024 * 1024 },

  updatedAt:  { type: Date, default: Date.now },
  deletedAt:  { type: Date, default: null },
}, { timestamps: true, strict: true })

blobSchema.index({ userId: 1, docType: 1, docId: 1 }, { unique: true })
blobSchema.index({ userId: 1, updatedAt: -1 })
```

**Six decisions:**

**`docId` is client-generated.** The server is not the source of truth for what a document *is*.
If the server minted ids, then two devices that each created "a note about my semester" could
never be reconciled, and offline-created work has nowhere to go. Client-generated UUIDs make
every device an equal peer.

**`byteLength` is stored even though it is not content.** You need it for quota checks and for
the sync cursor's "how much is coming" estimate. It leaks approximate document size and nothing
else — no filenames, no content types that could identify a document.

**`max: 10 * 1024 * 1024`.** A hard 10MB ceiling per blob, enforced in the schema. A scanned
Aadhaar at 3000 DPI is 15MB and you want a clear rejection, not a 15MB upload that OOMs the API
process and takes down every other user.

**`unique` on `(userId, docType, docId)`.** This is the upsert target. Two devices syncing the
same document concurrently resolve to one row, not two. Without it you get duplicate documents
and a user who has deleted something twice.

**`algo` and `kdfVersion` on every row.** Crypto agility. When you migrate from
`kdfVersion: 1` to Argon2id, existing rows are still readable and you can re-encrypt them in a
background job. **Without the version field, a crypto upgrade means the server cannot tell which
rows it can read** — because it cannot read any of them. You need the version to know what to
re-upload.

**No `filename`, no `mimeType`, no `label`.** It is tempting. Every one of those is PII —
`rohit_jadhav_aadhaar.pdf` identifies both the person and the document type. They belong in the
ciphertext, where they can be read by the user and not by you.

---

## Step 3 — `syncmeta` — the hot, queryable, content-free one

```ts
// apps/api/src/db/models/syncmeta.ts
const syncmetaSchema = new Schema({
  userId:   { type: Schema.Types.ObjectId, ref: "User", required: true },
  docId:    { type: String, required: true },
  docType:  { type: String, enum: ["fact", "document", "verse", "setlist"], required: true },

  /** Monotonic per user. The sync cursor. Gaps are fine; duplicates are not. */
  revision: { type: Number, required: true },

  /** Which device last wrote. For "changed on your phone" messaging. */
  deviceId: { type: Schema.Types.ObjectId, ref: "Device" },

  updatedAt: { type: Date, default: Date.now },

  /** ── Tombstone. The reason syncmeta is small and blobs is not. ── */
  deleted:   { type: Boolean, default: false },
  /** TTL index: tombstones self-destruct after 90 days. */
  purgeAfter: { type: Date, default: null },
}, { timestamps: true, strict: true })

syncmetaSchema.index({ userId: 1, revision: -1 })
syncmetaSchema.index({ userId: 1, docId: 1 }, { unique: true })
// TTL. MongoDB deletes expired docs on its own — no cron job.
syncmetaSchema.index({ purgeAfter: 1 }, { expireAfterSeconds: 0 })
```

### Why tombstones exist at all

Without a `deleted` flag in a separate collection, deletion is un-syncable:

```
Phone deletes a document.      → row removed from blobs
Laptop syncs.                 → no change to pull. Document is still there.
```

**Deleted documents resurrect forever.** Every device that was offline during the deletion keeps
a local copy and keeps uploading it, and you get a document reappearing three weeks after the
user deleted it. Users interpret that as "this thing has a copy of my Aadhaar I cannot get rid
of," and *they are right*.

The tombstone is a two-byte flag in a tiny document. It is the single highest-value field in
this chapter.

### Why `purgeAfter` + a TTL index instead of a cron job

MongoDB's TTL index deletes expired documents natively, on its own schedule, with no code
running. A cron job is another moving part that can fail silently and leave tombstones forever.

90 days is a deliberate choice. It must exceed your longest plausible offline window — a phone
in a drawer for a semester, a laptop that was never powered on — or a very old client will
resurrect the document. **When you change the tombstone window, bump a `syncVersion` so old
clients know to do a full re-pull.** Chapter 8.

### Why `syncmeta` must never grow a content field

Look at what is in it: ids, a number, a date, two booleans. You could read that whole collection
into RAM. It is not personal data.

The moment someone adds `lastKnownLabel: String` for a nicer "changed your CGPA" notification,
you have converted a content-free index into a searchable index of what every user's fields are
named — which is the entire profile, in plain text, in a hot collection. **Put the display name
inside the ciphertext and decrypt it client-side.** The notification costs one round trip.

---

## Step 4 — `devices`

```ts
// apps/api/src/db/models/device.ts
const deviceSchema = new Schema({
  userId:   { type: Schema.Types.ObjectId, ref: "User", required: true },
  label:    { type: String, required: true, maxlength: 60 },   // "Rohit's MacBook"
  platform: { type: String, enum: ["web", "chrome", "edge", "firefox", "safari", "android", "ios"], required: true },
  /** SHA-256 of the browser's device fingerprint. Abuse detection only. */
  fingerprint: { type: String, required: true },

  createdAt:  { type: Date, default: Date.now },
  lastSeenAt: { type: Date, default: Date.now },
  revokedAt:  { type: Date, default: null },
}, { timestamps: true, strict: true })

deviceSchema.index({ userId: 1, fingerprint: 1 }, { unique: true, partialFilterExpression: { revokedAt: null } })
```

> **Devices are what make "revoke this laptop" a real feature** rather than a suggestion.
> Without a device record, revoking a lost device means rotating the whole account and logging
> you out everywhere — including the phone you still have. With it, you revoke one row and every
> refresh token bound to it dies. Chapter 6.

**`label` is free-text from the user, which makes it log-worthy.** A user will name a device
"work laptop (boss can see)" and it will sit in your logs in plaintext. Your scrubber handles
it; do not skip it.

---

## Step 5 — `refreshtokens`

```ts
// apps/api/src/db/models/refreshtoken.ts
const refreshTokenSchema = new Schema({
  userId:   { type: Schema.Types.ObjectId, ref: "User", required: true },
  deviceId: { type: Schema.Types.ObjectId, ref: "Device", required: true },
  /** SHA-256. Storing the raw token means a DB leak is a session leak. */
  tokenHash: { type: String, required: true },

  createdAt: { type: Date, default: Date.now },
  expiresAt: { type: Date, required: true },
  revokedAt: { type: Date, default: null },
  /** Set on reuse. See the rotation-reuse detection in Chapter 6. */
  rotatedTo: { type: Schema.Types.ObjectId, ref: "RefreshToken", default: null },
}, { strict: true })

refreshTokenSchema.index({ tokenHash: 1 }, { unique: true })
refreshTokenSchema.index({ expiresAt: 1 }, { expireAfterSeconds: 0 })   // auto-purge
refreshTokenSchema.index({ userId: 1, deviceId: 1 })
```

**Four decisions:**

**`tokenHash`, never the token.** If someone reads your `refreshtokens` collection, hashed
tokens give them nothing. Raw tokens give them a year of access to every account in the
collection. This is the single most important line in the auth model.

**A TTL index on `expiresAt`.** MongoDB removes expired tokens. No cleanup job.

**`rotatedTo`.** This field is what makes reuse detection possible: if a token that was already
rotated is presented again, the token was stolen. Chapter 6 uses it to revoke the whole family.
Without it you have no theft signal.

**`expiresAt` defaults to 30 days; access tokens to 15 minutes.** The ratio matters. Short access
tokens mean the stolen-token window is small; long refresh tokens with rotation mean a stolen
refresh token is single-use and detectable.

---

## Step 6 — A note on every schema

```ts
{
  strict: true,          // reject fields not declared. THE most important flag.
  timestamps: true,      // createdAt + updatedAt, maintained by Mongoose
  sanitizeFilter: true,  // reject { $ne: null } style injection
}
```

**`strict: true` is the one to never disable.** With it off, Mongoose silently drops unknown
fields — which sounds safe and is not. It means a typo in a field name writes a document that
*looks* correct and has no data in it. A typo in `confindence` becomes a field called
`confindence`, the write succeeds, the value is gone, and you find out when the UI renders blank.

> **`sanitizeFilter: true` is set globally in `connect.ts` and per-schema here.** Set it in both
> places. The global one is your safety net; the per-schema one is your statement of intent. If a
> future contributor disables one, the other still holds.

---

## Step 7 — Cascade delete

```ts
// apps/api/src/db/models/cascade.ts
import { UserModel } from "./user"
import { BlobModel } from "./blob"
import { SyncMetaModel } from "./syncmeta"
import { DeviceModel } from "./device"
import { RefreshTokenModel } from "./refreshtoken"

/**
 * §18's right to be forgotten. Irreversible. Therefore it is one
 * function, called from one place, with no HTTP shortcut.
 */
export async function purgeUser(userId: string) {
  const s = mongoose.startSession()
  try {
    return await s.withTransaction(async () => {
      // Documents first, then routing, then identity.
      // If this fails partway, orphan blobs are cheaper than
      // orphan tokens.
      await BlobModel.deleteMany({ userId })
      await SyncMetaModel.deleteMany({ userId })
      await RefreshTokenModel.deleteMany({ userId })
      await DeviceModel.deleteMany({ userId })
      return UserModel.deleteOne({ _id: userId })
    })
  } finally {
    await s.endSession()
  }
}
```

> **Order is documents → routing → identity, and it is deliberate.** If the transaction fails
> halfway, you want the failure mode to be "orphaned blobs" rather than "orphaned refresh
> tokens." Orphan blobs cost storage. Orphan tokens mean an account that cannot be logged out.
>
> **Transactions require a replica set.** `mongodb://localhost:27017` standalone does **not**
> support them. Your local Docker instance needs `--replSet rs0` plus
> `rs.initiate()`, or you get
> `Transaction numbers are only allowed on a replica set member or mongos`. Atlas has this by
> default; local does not, and this bites everyone once.

---

## Step 8 — Inspect what you actually built

Do this before writing a single route. It is the fastest way to catch a content field that
slipped into `syncmeta`.

```ts
// apps/api/src/db/seed.ts
await connectDb()
await UserModel.deleteMany({})
const u = await UserModel.create({
  email: "dev@refrain.local",
  authHash: await argon2.hash("dev-auth-key"),
})
await BlobModel.create({
  userId: u._id, docId: "fact-1", docType: "fact",
  wrappedDek: { ciphertext: Buffer.from("x"), iv: Buffer.from("y") },
  ciphertext: Buffer.from("opaque"), iv: Buffer.from("z"),
  byteLength: 6,
})
await SyncMetaModel.create({ userId: u._id, docId: "fact-1", docType: "fact", revision: 1 })
console.log("seeded")
```

```bash
# THE AUDIT. Run this on every staging deploy.
mongosh refrain --quiet --eval '
  db.blobs.aggregate([{ $project: { _id: 0, hasLabel: { $ifNull: ["$label", "MISSING"] } } }]).toArray()
'
```

Or, faster and better — write a script that fails CI:

```ts
// apps/api/src/db/assert-no-content.test.ts
const SYNC_CONTENT_FIELDS = [
  "label", "title", "name", "filename", "value", "mimeType",
  "email", "phone", "address", "cgpa", "pan", "aadhaar",
]

describe("syncmeta holds no personal content", () => {
  it("declares no content fields", () => {
    const declared = Object.keys(SyncMetaModel.schema.paths)
    for (const f of SYNC_CONTENT_FIELDS) {
      expect(declared).not.toContain(f)
    }
  })

  it("is small enough to keep hot", () => {
    const paths = Object.keys(SyncMetaModel.schema.paths)
    expect(paths.length).toBeLessThanOrEqual(8)
  })
})
```

> **That second test is the clever one.** A hard cap of 8 fields turns "no personal content"
> from a review convention into a build failure. Someone adding a ninth field has to change a
> test that says *this collection is deliberately tiny because it is deliberately readable by
> the server.* That is a conversation you want them to have.

---

## Step 9 — Commit

```bash
git add -A
git commit -m "feat(api): hybrid storage — blobs (opaque) + syncmeta (content-free), tombstones, DEK wrapping"
```

Add three rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | Hybrid storage: opaque `blobs` + content-free `syncmeta` | A server that cannot read your data cannot query your data. The split keeps sync fast without giving up encryption. |
| 2026-10-XX | Per-document DEK wrapped by `k_vault` | Random per-document keys mean one leak is one document, and selective sync becomes "do not upload it." |
| 2026-10-XX | Tombstones with a 90-day TTL | Without a deletion flag, deleted documents resurrect from any offline device forever. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The key hierarchy and every arrow in that diagram** | **You.** Everything depends on it |
| **`deriveVaultKey` / `deriveAuthKey` / `wrapDEK` / `unwrapDEK`** | **You.** Never paste key derivation |
| **Why `syncmeta` must stay tiny** | **You** |
| **`assert-no-content.test.ts` and the 8-field cap** | **You** |
| Tombstones and the resurrect bug | **You** |
| `max: 10MB` on the blob size | **You** — it is a capacity decision |
| `partialFilterExpression` on the email index | **You**, after reading the re-register bug |
| Mongoose schema boilerplate for the six models | **OpenCode** from these templates |
| `purgeUser` and the delete ordering | **OpenCode**, then verify the ordering reasoning yourself |
| `seed.ts` | **OpenCode** |
| Index definitions | **OpenCode**, then justify each one out loud |

---

## Gotchas in this chapter

**`Transaction numbers are only allowed on a replica set member`.** Local MongoDB is
standalone. Run it with `--replSet rs0` and `rs.initiate()`, or use Atlas for dev.

**`authHash` came back in a response body.** You forgot `select: false`, or a `.lean()` bypassed
it. Re-check with `curl` — never trust the schema alone.

**A user cannot re-register with a deleted email.** Your unique index has no
`partialFilterExpression`. See Step 1.

**`$where` and `$ne` injection returns the first user.** `sanitizeFilter` is off on that schema.
Set it globally *and* per-schema.

**An Aadhaar PDF is 15MB and the upload OOMs the API.** Raise `max` deliberately or reject with
a clear message telling the user to re-scan at lower resolution. Do not just raise the limit.

**Deleted documents keep coming back.** No tombstones. See Step 3.

**`syncmeta` is 200MB and queries got slow.** Someone added content fields. Run the audit in
Step 8.

**You cannot decrypt a `wrappedDek` on the server.** That is correct and it is the design. If
you were expecting to, re-read the diagram — the answer is that you cannot and should not.

**Argon2 on a 6-character test passphrase takes 400ms per hash in CI.** Set a lower cost factor
in `NODE_ENV=test` only. Never lower it in production.

---

## Verify before moving on

- [ ] All six models compile with `strict: true`
- [ ] `users.authHash` is `select: false` and absent from every default query
- [ ] Unique index on `users.email` has `partialFilterExpression`
- [ ] Unique index on `(userId, docType, docId)` in `blobs`
- [ ] TTL index on `syncmeta.purgeAfter` actually deletes a document after its date
- [ ] TTL index on `refreshtokens.expiresAt` deletes an expired token
- [ ] `purgeUser` removes rows from all six collections
- [ ] A transaction works against your local replica set
- [ ] `assert-no-content.test.ts` passes — and you can name all 8 `syncmeta` fields
- [ ] No `filename`, `mimeType`, or `label` anywhere in `blobs` or `syncmeta`
- [ ] `k_vault` is `extractable: false`; `dek` is `true`
- [ ] `deriveVaultKey` and `deriveAuthKey` produce different keys from the same root

---

## Check yourself before Chapter 6

1. **Why are `blobs` and `syncmeta` separate collections rather than one?**
2. **What does the server hold that lets it authenticate you without decrypting you?**
3. **Why is `syncmeta` allowed to be readable, and what does "readable" permit exactly?**
4. **What breaks if you delete a row from `blobs` instead of setting a tombstone?**
5. **Why does a per-document DEK beat one shared key?**
6. **Why is `dek` extractable while `k_vault` is not?**
7. **Why does `refreshTokenSchema` store `tokenHash` and not the token?**
8. **Why does the `partialFilterExpression` on the email index matter six months later?**
9. **What is the 8-field cap in that test protecting, and what happens when someone violates it?**

---

**Next: [Chapter 6 — Auth](./06-auth.md)** — registration with the HKDF split, Argon2id,
short-lived JWTs, refresh rotation with reuse detection, and revoking one device without
logging out the rest.