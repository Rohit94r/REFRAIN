# Chapter 12 — The Vault

> **Day 12 · Goal: encrypted, searchable, provenance-carrying profile. Lock and unlock work.**
>
> This is the foundation everything else stands on. If provenance here is sloppy, the review
> screen lies, and the whole product is worthless.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **Vault** — your encrypted storage on your own device.
- **Encryption** — turning readable text into unreadable text that only your key
  can reverse.
- **AES-GCM** — the encryption used here. It encrypts *and* detects tampering.
- **Ciphertext** — the unreadable encrypted result.
- **IV / nonce** — a random number used once with a key. Reusing it breaks
  AES-GCM completely.
- **Key** — the secret that makes encryption reversible.
- **Non-extractable key** — a key the browser will not hand out as raw bytes,
  so a bug cannot leak it.
- **PBKDF2** — the slow function that turns your passphrase into a key.
- **Key hierarchy** — splitting one root key into separate keys for separate
  jobs.
- **HKDF** — the function that splits one key into several.
- **IndexedDB / Dexie** — the browser's built-in database, used to store the
  vault.
- **Tamper detection** — knowing when encrypted data has been changed.

---

## Understand this first

### Why encryption is not the feature — provenance is

You could ship without encryption and nobody would notice on day one. **The thing that makes
you credible is the small grey text on every row: `from marksheet.pdf · p.2 · 4 Aug`.**

Consider what you are actually asking a 19-year-old to do. They are about to upload their
10th marksheet, their Aadhaar, their father's PAN. To a tool made by one person, with no
company behind it, no SOC2, no audit. And then they are about to trust it with answers on
scholarship forms.

The only reason that trade is acceptable is if **the tool shows its work at all times.** The
encryption keeps the data safe if their laptop is stolen. The provenance makes the data
*trustworthy*. Only the second one is defensible in a sentence you can say out loud.

So: build the crypto properly because you must, but understand that the crypto is not why
anyone will use this. The provenance is.

### Why "encrypted at rest" is a weaker claim than it sounds

IndexedDB is a file on disk. If you encrypt the *values*, but leave the *keys* in a
`localStorage` entry, you have encrypted nothing — anyone with the file reads both.

The chain has to be unbroken end to end:

```
passphrase (typed by the human, never stored)
   ↓ PBKDF2-SHA256, 600,000 iterations
   ↓ + a random salt stored in plaintext
   ↓ + a random IV stored with each record
AES-256-GCM key (held in memory only, cleared on lock)
   ↓
encrypted records in IndexedDB
```

**Three things must never touch disk:** the passphrase, the derived key, and the plaintext
values. Your `.gitignore` is the first line; this chain is the second.

> `600,000 iterations` is OWASP's recommendation for PBKDF2-HMAC-SHA256. It takes about a
> quarter of a second on a laptop — which is a feature. A login that takes 250ms feels
> considered. A login that takes 4ms feels like nothing is being protected.

### Why Argon2id exists, and why you are not using it yet

Argon2id is memory-hard. An attacker who copies your encrypted database has to spend RAM,
not just CPU, to attack it. PBKDF2 is CPU-hard only, so GPU attacks are cheap.

Argon2id needs WASM. That is a dependency, a bundle-size cost, and a correctness risk you
do not want in your first week. **Ship PBKDF2 now, note Argon2id in §19 as a planned swap
with a versioned KDF id so old vaults can be re-encrypted.** That is the right engineering
order: correct now, better later, and no user loses data in the migration.

### Why the profile is a graph and not a schema

Google Forms asks for `10th Percentage`. A bank KYC asks for `Class X aggregate`. An
internship application asks for `Secondary School %`.

If you model the profile as fixed columns, you will spend the next month adding columns and
still fail on field 40. Instead model it as a **small set of canonical facts plus freeform
attributes**, so the mapping layer (Chapter 14) does the translation instead of your schema.

```ts
// Fixed: the ~20 things every form eventually asks for
{ name, email, phone, dateOfBirth, address, cgpa, ... }

// Open: everything else
attributes: { "10th_percentage": { value, provenance }, "class_x_aggregate": { ... } }
```

The cost is that you must write the mapping layer. That layer is Chapter 14, and it is the
moat. You cannot avoid it by choosing a nicer schema.

### The unlock model is a UX decision, not a security one

Do you unlock once per session, once per day, or on every action?

| Model | Friction | Risk | Recommendation |
|---|---|---|---|
| Once per session | Low | Your unlocked vault is readable by any XSS on the page | ✅ **Use this.** `chrome.storage.session` |
| Once per day | Lower | Worse — a browser left open for a day | No |
| Every action | Insane | Users will disable it, then disable the extension | No |

**Once per browser session**, key in memory only, cleared when Chrome quits or the service
worker sleeps. Combined with a short idle timeout (15 min) if you want to be stricter.

---

## Step 1 — The crypto module (write this yourself, carefully)

`packages/vault/src/crypto/`

```ts
// packages/vault/src/crypto/kdf.ts

export const KDF_ID = "pbkdf2-sha256" as const
export const KDF_ITERATIONS = 600_000
const KEY_LENGTH_BITS = 256

export interface KdfParams {
  id: typeof KDF_ID
  iterations: number
  salt: Uint8Array
}

export const newSalt = (): Uint8Array => crypto.getRandomValues(new Uint8Array(16))

/**
 * PBKDF2 is memory-hard: NO. It is CPU-hard only. See the note in
 * the chapter — Argon2id is the planned upgrade, gated on KDF_ID
 * so old vaults can be re-derived without data loss.
 */
export async function deriveKey(passphrase: string, params: KdfParams): Promise<CryptoKey> {
  const base = await crypto.subtle.importKey(
    "raw",
    new TextEncoder().encode(passphrase),
    "PBKDF2",
    false,
    ["deriveKey"],
  )

  return crypto.subtle.deriveKey(
    {
      name: "PBKDF2",
      hash: "SHA-256",
      salt: params.salt as BufferSource,
      iterations: params.iterations,
    },
    base,
    { name: "AES-GCM", length: KEY_LENGTH_BITS },
    false, // ← non-extractable. The key cannot be read back out.
    ["encrypt", "decrypt"],
  )
}
```

> **`extractable: false` is not decoration.** It makes it structurally impossible for your
> own code to accidentally serialise the key into `localStorage`. The browser refuses.
> This is the difference between "I meant not to" and "it cannot happen".

```ts
// packages/vault/src/crypto/aead.ts

export interface Sealed {
  ciphertext: ArrayBuffer
  iv: Uint8Array
}

/**
 * AES-256-GCM. Authenticated: if anyone flips a bit in the ciphertext,
 * decryption throws. Tamper-evidence is free here — do not add your own.
 */
export async function seal(key: CryptoKey, plaintext: unknown): Promise<Sealed> {
  const iv = crypto.getRandomValues(new Uint8Array(12)) // 96-bit IV, the GCM standard
  const data = new TextEncoder().encode(JSON.stringify(plaintext))
  const ciphertext = await crypto.subtle.encrypt({ name: "AES-GCM", iv: iv as BufferSource }, key, data as BufferSource)
  return { ciphertext, iv }
}

export async function unseal<T>(key: CryptoKey, sealed: Sealed): Promise<T> {
  const buf = await crypto.subtle.decrypt(
    { name: "AES-GCM", iv: sealed.iv as BufferSource },
    key,
    sealed.ciphertext,
  )
  return JSON.parse(new TextDecoder().decode(buf)) as T
}
```

**A fresh IV per record, never reused.** Reusing an IV with the same key under GCM is
catastrophic — it leaks the authentication key. `crypto.getRandomValues` inside `seal` makes
reuse structurally impossible, which is exactly the kind of bug you want to be structurally
impossible about.

```ts
// packages/vault/src/crypto/random.ts

/** Human-readable recovery phrase. 6 groups of 4 from a 32-symbol alphabet. */
const ALPHABET = "ABCDEFGHJKLMNPQRSTUVWXYZ23456789" // no I, O, 0, 1 — transcription errors

export function generateRecoveryPhrase(groups = 6, size = 4): string {
  const bytes = crypto.getRandomValues(new Uint8Array(groups * size))
  return Array.from({ length: groups }, (_, g) =>
    Array.from(bytes.slice(g * size, (g + 1) * size))
      .map((b) => ALPHABET[b % ALPHABET.length])
      .join(""),
  ).join("-")
}
```

> **The alphabet matters.** I, O, 0 and 1 are removed because a student will read this off a
> printed card at 2am. `0/O` ambiguity turns into "I cannot recover my vault" and there is
> no support desk. Ambiguous glyphs are a data-loss bug.

### Test the failure path, not just the happy path

```ts
import { describe, it, expect } from "vitest"
import { deriveKey, newSalt, KDF_ITERATIONS } from "./kdf"
import { seal, unseal } from "./aead"

describe("vault crypto", () => {
  it("round-trips a value", async () => {
    const key = await deriveKey("correct horse", { id: "pbkdf2-sha256", iterations: 1000, salt: newSalt() })
    const s = await seal(key, { cgpa: "8.7" })
    expect(await unseal(key, s)).toEqual({ cgpa: "8.7" })
  })

  it("fails to open with the wrong passphrase", async () => {
    const salt = newSalt()
    const a = await deriveKey("right", { id: "pbkdf2-sha256", iterations: 1000, salt })
    const b = await deriveKey("wrong", { id: "pbkdf2-sha256", iterations: 1000, salt })
    const s = await seal(a, { secret: 1 })
    await expect(unseal(b, s)).rejects.toThrow()
  })

  it("detects a flipped ciphertext bit", async () => {
    const key = await deriveKey("pw", { id: "pbkdf2-sha256", iterations: 1000, salt: newSalt() })
    const s = await seal(key, { cgpa: "8.7" })
    const bytes = new Uint8Array(s.ciphertext)
    bytes[0] = bytes[0]! ^ 0x01
    await expect(unseal(key, { ...s, ciphertext: bytes.buffer })).rejects.toThrow()
  })

  it("never produces the same IV twice", async () => {
    const key = await deriveKey("pw", { id: "pbkdf2-sha256", iterations: 1000, salt: newSalt() })
    const ivs = new Set<string>()
    for (let i = 0; i < 50; i++) {
      const s = await seal(key, { i })
      ivs.add(btoa(String.fromCharCode(...s.iv)))
    }
    expect(ivs.size).toBe(50)
  })

  it("takes a real amount of time at production iteration count", async () => {
    const t0 = performance.now()
    await deriveKey("pw", { id: "pbkdf2-sha256", iterations: KDF_ITERATIONS, salt: newSalt() })
    expect(performance.now() - t0).toBeGreaterThan(100)
  })
})
```

```bash
pnpm --filter @refrain/vault test
```

The last test is worth keeping permanently. If someone later drops iterations to 1,000 to
"make login faster," the suite fails and you find out in CI instead of in a security report.

---

## Step 2 — The database

```bash
pnpm --filter @refrain/vault add dexie
```

```ts
// packages/vault/src/db/schema.ts

import { Dexie, type Table } from "dexie"

export interface ProfileFact {
  /** "name", "email", "cgpa", or an attribute key like "10th_percentage" */
  key: string
  value: string
  /** Pillar 1. Non-optional. A fact without provenance is not a fact. */
  provenance: Provenance
  /** Highest confidence value seen for this key. */
  confidence: number
  updatedAt: number
}

export interface DocumentRecord {
  id: string
  kind: "marksheet" | "resume" | "pan" | "aadhaar" | "transcript" | "other"
  /** Filename only. The file blob is encrypted separately, never this row. */
  filename: string
  /** PII is never stored. Just enough to re-prompt the user if needed. */
  mimeType: string
  byteLength: number
  addedAt: number
}

export interface EncryptedBlobRow {
  id: string
  recordId: string
  sealed: { ciphertext: ArrayBuffer; iv: Uint8Array }
}

/** Plaintext. Holds no secrets — only the salt needed to re-derive the key. */
export interface VaultMeta {
  id: "meta"
  kdf: { id: string; iterations: number; salt: Uint8Array }
  recoveryPhraseHash: string
  createdAt: number
}

export class RefrainDB extends Dexie {
  facts!: Table<ProfileFact, string>
  documents!: Table<DocumentRecord, string>
  blobs!: Table<EncryptedBlobRow, string>
  meta!: Table<VaultMeta, string>

  constructor() {
    super("refrain")
    this.version(1).stores({
      facts: "key, updatedAt, provenance.kind",
      documents: "id, kind, addedAt",
      blobs: "id, recordId",
      meta: "id",
    })
  }
}

export const db = new RefrainDB()
```

**Read `this.version(1).stores` carefully.** It is not a schema — it is an *index* declaration.
IndexedDB stores are dumb key-value stores; Dexie turns these strings into real indexes.
`provenance.kind` as a compound-ish index lets you answer "show me everything from documents"
without a full scan, which is what the provenance inspector in Chapter 15 needs.

### IndexedDB is asynchronous and that is not a detail

```ts
const rows = await db.facts.toArray()   // ✅
const rows = db.facts.toArray()         // ❌ returns a Promise, `.map` fails
```

Every read and write is `await`. If you find yourself wanting synchronous access — "just give
me the profile" — that is a design smell. It means you have put UI-blocking work in the hot
path. Read once into a Zustand store at unlock.

---

## Step 3 — The session

```ts
// packages/vault/src/session.ts

import { db } from "./db/schema"
import { deriveKey, KDF_ITERATIONS, KDF_ID, newSalt } from "./crypto/kdf"

const IDLE_TIMEOUT_MS = 15 * 60 * 1000

class VaultSession {
  #key: CryptoKey | null = null
  #timer: ReturnType<typeof setTimeout> | null = null

  get isUnlocked() { return this.#key !== null }
  get isReady() { return db.meta.get("meta") !== undefined }

  async setup(passphrase: string): Promise<void> {
    const salt = newSalt()
    const kdf = { id: KDF_ID, iterations: KDF_ITERATIONS, salt }
    this.#key = await deriveKey(passphrase, kdf)
    await db.meta.put({ id: "meta", kdf, recoveryPhraseHash: "", createdAt: Date.now() })
  }

  async unlock(passphrase: string): Promise<boolean> {
    const meta = await db.meta.get("meta")
    if (!meta) return false
    // Guard against a vault created by an older KDF version.
    const iterations = (meta.kdf as { iterations?: number }).iterations ?? KDF_ITERATIONS
    const candidate = await deriveKey(passphrase, { ...meta.kdf, iterations })
    // A sentinel record is the only honest way to test a passphrase:
    // there is no ciphertext to try until you have written one.
    const probe = await db.blobs.get("__probe__")
    if (!probe) { this.#key = candidate; this.#arm(); return true }
    try {
      await crypto.subtle.decrypt(
        { name: "AES-GCM", iv: probe.sealed.iv as BufferSource },
        candidate,
        probe.sealed.ciphertext,
      )
      this.#key = candidate
      this.#arm()
      return true
    } catch {
      return false // ← never distinguish "wrong passphrase" from "corrupt" in the UI copy
    }
  }

  lock() {
    this.#key = null
    if (this.#timer) clearTimeout(this.#timer)
    this.#timer = null
  }

  #arm() {
    if (this.#timer) clearTimeout(this.#timer)
    this.#timer = setTimeout(() => this.lock(), IDLE_TIMEOUT_MS)
  }

  private require(): CryptoKey {
    if (!this.#key) throw new VaultLockedError()
    return this.#key
  }

  /** Generic sealed read/write. The session owns the key; nobody else sees it. */
  async write<T>(table: "facts", key: string, value: T): Promise<void> {
    const sealed = await seal(this.require(), value)
    await (db[table] as Table<T, string>).put({ ...(value as object), ...sealed } as never)
  }
}

export class VaultLockedError extends Error {
  constructor() { super("vault is locked") }
}

export const vault = new VaultSession()
```

### Three decisions worth arguing about

**`#key` is a class private field, not a module variable.** It cannot be read from outside,
not even by accident, not even by a debugger that walks the module scope. It is a
language-level guarantee rather than a naming convention.

**A `__probe__` record.** You cannot verify a passphrase without something to decrypt. The
alternative — decrypting the first real record — works too, but fails confusingly on an empty
vault. One extra sealed blob makes unlock total.

**The error message is deliberately vague.** "Wrong passphrase or corrupted vault." A precise
"wrong passphrase" is friendlier; a precise "corrupted vault" tells an attacker the salt is
valid. The ambiguity costs one support ticket and buys you not being a padding oracle.

**`lock()` must be called on every `chrome.storage.session` clear and before the extension
service worker dies.** Add that to Chapter 13.

---

## Step 4 — The profile graph and provenance

```ts
// packages/vault/src/profile.ts

import type { Provenance, Confidence } from "@refrain/fields"
import { REVIEW_THRESHOLD } from "@refrain/fields"

export interface Profile {
  /** Canonical facts every form eventually asks for. */
  core: {
    fullName?: ProfileFact
    email?: ProfileFact
    phone?: ProfileFact
    dateOfBirth?: ProfileFact
    address?: ProfileFact
    city?: ProfileFact
    state?: ProfileFact
    pincode?: ProfileFact
    cgpa?: ProfileFact
    tenthPercentage?: ProfileFact
    twelfthPercentage?: ProfileFact
    university?: ProfileFact
    degree?: ProfileFact
    graduationYear?: ProfileFact
    pan?: ProfileFact
    aadhaar?: ProfileFact
  }
  /** Everything else. Chapter 14 learns to read this. */
  attributes: Record<string, ProfileFact>
  documents: DocumentRecord[]
}

export interface ProfileFact {
  value: string
  provenance: Provenance
  confidence: number
  updatedAt: number
}

/** Read a fact by dotted path, tolerating absence. Never throws. */
export function readPath(profile: Profile, path: string): ProfileFact | undefined {
  const parts = path.split(".")
  let node: unknown = profile.core
  for (const p of parts) {
    if (node == null || typeof node !== "object") return undefined
    node = (node as Record<string, unknown>)[p]
  }
  return node as ProfileFact | undefined
}

/** Flatten to the shape the mapping engine consumes. Pure, synchronous, testable. */
export function indexFacts(profile: Profile): Map<string, ProfileFact> {
  const out = new Map<string, ProfileFact>()
  for (const [key, fact] of Object.entries(profile.core)) {
    if (fact) out.set(key, fact)
  }
  for (const [key, fact] of Object.entries(profile.attributes)) out.set(key, fact)
  return out
}

/**
 * The single gate. If a fact's confidence is below threshold it must not
 * reach a form. Centralised so there is exactly one place to audit.
 */
export function isSafeToFill(fact: ProfileFact | undefined): fact is ProfileFact & { confidence: number } {
  return fact !== undefined && fact.confidence >= REVIEW_THRESHOLD
}
```

`readPath` and `indexFacts` are **pure functions with no I/O.** Write their tests first. They
are the seam where Chapter 14's mapping engine gets tested without a browser, without Dexie,
and without a form. That is not a detail — it is why you can test the moat in 40ms.

---

## Step 5 — Documents: metadata now, bytes later

Do not write the document extraction pipeline in this chapter. But **do** write the storage
shape, because retrofitting encryption onto uploaded files after users have data is a
migration you do not want.

```ts
// packages/vault/src/documents.ts

import { db } from "./db/schema"
import { seal, unseal } from "./crypto/aead"
import { vault } from "./session"

export async function storeDocument(
  file: File,
  kind: DocumentRecord["kind"],
  key: CryptoKey,
): Promise<DocumentRecord> {
  const record: DocumentRecord = {
    id: crypto.randomUUID(),
    kind,
    filename: file.name,
    mimeType: file.type,
    byteLength: file.size,
    addedAt: Date.now(),
  }
  // Seal the bytes, not the metadata. The metadata row holds no PII beyond the filename.
  const sealed = await seal(key, await file.arrayBuffer())
  await db.documents.add(record)
  await db.blobs.add({ id: record.id, recordId: record.id, sealed })
  return record
}

export async function readDocument(id: string, key: CryptoKey): Promise<Blob> {
  const row = await db.blobs.get(id)
  if (!row) throw new Error("document not found")
  const buf = await unseal<ArrayBuffer>(key, row.sealed)
  return new Blob([buf], { type: (await db.documents.get(id))?.mimeType ?? "application/octet-stream" })
}
```

> **Never `JSON.stringify` a PDF.** `seal` calls `JSON.stringify` internally, which is
> correct for your profile objects and catastrophic for a 4MB binary blob — it will corrupt
> it and blow your memory budget twice over. If you ever need to seal raw bytes, add a
> dedicated `sealBytes` that skips the stringify. Chapter 17 uses it for the extractor.

---

## Step 6 — The unlock screen

This is the first real UI you build, and it teaches you something about the whole product:
**a passphrase prompt is a trust screen.** Do not treat it as a form field.

```tsx
// apps/web/src/routes/Unlock.tsx
import { useState } from "react"
import { vault } from "@refrain/vault"
import { Mascot } from "@refrain/ui"

export function Unlock() {
  const [value, setValue] = useState("")
  const [error, setError] = useState<string | null>(null)
  const [busy, setBusy] = useState(false)

  async function onSubmit(e: React.FormEvent) {
    e.preventDefault()
    setBusy(true); setError(null)
    const ok = await vault.unlock(value)
    setBusy(false)
    if (!ok) {
      setError("That passphrase did not open this vault.")
      setValue("")
    }
  }

  return (
    <main className="min-h-dvh grid place-items-center bg-surface p-8">
      <form onSubmit={onSubmit} className="w-full max-w-sm rounded-panel bg-surface-muted p-8 shadow-soft">
        <Mascot state="idle" />
        <h1 className="mt-6 text-xl font-semibold text-ink">Your vault is locked</h1>
        {/* §11: say the thing that matters, before they type. */}
        <p className="mt-2 text-sm text-ink-muted">
          Nothing leaves this device. Your passphrase is never sent anywhere, not even to us.
        </p>

        <input
          type="password"
          value={value}
          onChange={(e) => setValue(e.target.value)}
          placeholder="Passphrase"
          aria-label="Vault passphrase"
          className="mt-6 w-full rounded-card border bg-surface px-4 py-3 text-ink"
          // ← No autocomplete hint. Password managers must NOT fill this. See below.
          autoComplete="off"
        />
        {error && <p role="alert" className="mt-2 text-sm text-prov-lowconf">{error}</p>}

        <button disabled={busy || value.length < 8}
          className="mt-4 w-full rounded-card bg-brand-500 py-3 font-medium text-white disabled:opacity-50">
          {busy ? "Opening…" : "Unlock"}
        </button>
      </form>
    </main>
  )
}
```

Three things here that are not obvious:

**`autoComplete="off"` is load-bearing.** Browsers will happily save your vault passphrase in
their own password store and sync it to Google or iCloud. That would mean your encryption
claim is technically true and practically false. Verify the behaviour in DevTools →
Application → Autofill before you ship.

**`disabled` when `value.length < 8`.** Do not silently correct someone else's passphrase
choice — warn, do not forbid. This 8 is a floor on the *recommendation*, not a rule you
enforce on their data.

**The reassurance is above the input, not below.** People type first and read second. The
privacy line has to land before the field, or it lands never.

---

## Step 7 — Bridging the vault into React

`VaultSession` is a plain class outside React, holding a private `CryptoKey`. React needs to
know when it changes. **Do not poll, and do not mirror the vault into a Zustand store** — two
sources of truth for one lock state is a bug you will spend an afternoon on.

```ts
// packages/vault/src/session.ts — add to VaultSession
type Listener = () => void

class VaultSession {
  #key: CryptoKey | null = null
  #timer: ReturnType<typeof setTimeout> | null = null
  #listeners = new Set<Listener>()

  get isUnlocked() { return this.#key !== null }

  // getSnapshot MUST return a cached value. React calls it on every
  // render and bails out if the value is identity-equal.
  #snapshot: VaultSnapshot = { unlocked: false, ready: false, revision: 0 }

  getSnapshot = (): VaultSnapshot => this.#snapshot

  subscribe = (fn: Listener): (() => void) => {
    this.#listeners.add(fn)
    return () => this.#listeners.delete(fn)
  }

  #notify() {
    this.#snapshot = {
      unlocked: this.isUnlocked,
      ready: this.isReady,
      revision: this.#snapshot.revision + 1,
    }
    for (const fn of this.#listeners) fn()
  }
}
```

```tsx
// packages/vault/src/react.ts — the one hook every component uses
import { useSyncExternalStore } from "react"
import { vault, type VaultSnapshot } from "./session"

export function useVault(): VaultSnapshot {
  return useSyncExternalStore(vault.subscribe, vault.getSnapshot, vault.getSnapshot)
}
```

> **Why the cached `#snapshot` is mandatory.** `useSyncExternalStore` compares the previous
> return value with `Object.is` and silently does nothing if they match. If `getSnapshot`
> returns a fresh `{ unlocked: true }` object literal every call, React sees a "new" snapshot on
> every render, re-renders forever, and eventually throws *"The result of getSnapshot should be
> cached to avoid an infinite loop."* You must mutate the cached object, bump a `revision`
> counter, and notify — which is exactly what the code above does.
>
> **`revision` is not decoration.** `unlocked` alone is a boolean that flips twice
> (`false → true → false`) and is fine. But you will add `status: "idle" | "deriving"` later
> and find that two transitions produce the same boolean. The revision guarantees a change is
> always a change.

Now `UnlockGate` is trivial and correct:

```tsx
import { useVault } from "@refrain/vault/react"

export function UnlockGate() {
  const { ready, unlocked } = useVault()

  if (!ready)  return <Onboarding />
  if (!unlocked) return <Unlock />

  return <Outlet />
}
```

**Call `#notify()` at exactly three places:** successful `setup()`, successful `unlock()`,
and `lock()`. Miss the third and your sidebar keeps claiming "Vault unlocked" after a 15-minute
idle timeout. That is a security-relevant lie in a security UI.

> **Clear the key on lock, then notify.** Never notify first. If React renders "unlocked"
> while the key is still in memory, you have a window where the UI claims a state that is no
> longer true.

---

## Step 8 — Commit

```bash
git add -A
git commit -m "feat(vault): AES-GCM + PBKDF2(600k), sealed session, profile graph, React bridge"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`crypto/kdf.ts`, `crypto/aead.ts`, `crypto/random.ts`** | **You. Entirely. Never paste crypto.** |
| The five crypto tests, especially the tamper and IV-uniqueness cases | **You** |
| `VaultSession` with `#key` private field and idle lock | **You** |
| Dexie `stores` index declaration | **You** — you will extend it |
| `readPath`, `indexFacts`, `isSafeToFill` | **You.** Pure functions are the seam you will test for two weeks |
| `Unlock.tsx` styling | **OpenCode** — you write the copy, it writes the classNames |
| `storeDocument` / `readDocument` | **OpenCode** |
| `sealBytes` for raw binary (needed in Chapter 17) | **OpenCode**, then you review the byte handling |
| A test for a password manager autofilling the passphrase | **OpenCode** — it needs DevTools knowledge you will not have yet |

---

## Gotchas in this chapter

**`crypto.subtle` is undefined.** It requires a secure context — `https://` or
`http://localhost`. `file://` will not work. This is also why the extension needs a real
host permission rather than running from a folder.

**"Uncaught OperationError" on decrypt and nothing else.** Wrong passphrase, or a tampered
ciphertext. GCM gives you no detail by design. Log which one you intended to test.

**Everything is black after reload.** You stored the key in a module variable and the service
worker restarted. That is correct behaviour — see the session model. Handle `isUnlocked === false`
in your root component.

**Unlock takes 6 seconds.** You are running on a low iteration count from a test, or the
browser is throttling because the tab is backgrounded. Check you are not still using `1000`.

**`Uint8Array` passed to `deriveKey` throws.** Wrap in `BufferSource` or cast. TS 5.7+
tightened this. `salt as BufferSource` is the pragmatic fix.

**The key got extracted somewhere.** Search for `exportKey`, `structuredClone` on a CryptoKey,
or a `postMessage` of the session object. A `CryptoKey` that crosses a boundary by accident
becomes structured-cloneable and no longer non-extractable.

**Chrome's IndexedDB quota.** Default is a percentage of disk, which is usually fine. But a
student on a 32GB laptop with a full drive will hit it. Handle `QuotaExceededError` with a
real message, not a silent catch.

---

## Verify before moving on

- [ ] All five crypto tests pass, including tamper detection and IV uniqueness
- [ ] Production iteration count is 600,000 and unlock takes under 500ms
- [ ] `devtools` → Application → Autofill: the passphrase is **not** offered for saving
- [ ] Wrong passphrase shows a vague message, no stack trace, no partial state
- [ ] `vault.lock()` clears the key and the UI immediately reflects it
- [ ] Profile survives reload after unlock, still encrypted on disk
- [ ] Opening IndexedDB in DevTools shows only ciphertext
- [ ] `readPath` / `indexFacts` / `isSafeToFill` have unit tests, no I/O
- [ ] Recovery phrase generates 6 groups and never contains I/O/0/1

---

## Check yourself before Chapter 13

1. **Why does `deriveKey` use `extractable: false`, and what breaks if it is `true`?**
2. **Why a fresh IV per record, and what happens if you reuse one under GCM?**
3. **Why is the wrong-passphrase message deliberately vague?**
4. **What happens to your key when the extension service worker sleeps? Why is that correct?**
5. **Why does `isSafeToFill` live in the vault and not in the review screen?**
6. **You find `autoComplete="off"` does not stop Safari from offering to save. What do you do?**
7. **Why is the profile a graph with open attributes instead of a fixed schema?**

---

**Next: [Chapter 13 — The Content Script](./13-the-content-script.md)** — WXT, the MV3 manifest,
`all_frames`, form schema extraction, hidden-input drivers, and the native setter that stops
React from silently eating your values.