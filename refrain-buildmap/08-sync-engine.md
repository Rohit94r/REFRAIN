# Chapter 8 — The Sync Engine

> **Day 8 · Goal: two devices converge, offline edits survive, and no edit is ever
> silently lost.**
>
> The server cannot merge your data, because it cannot read your data. **Every line of conflict
> resolution logic therefore lives on the client.** That single constraint determines the whole
> design, starting with document granularity.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **Sync** — copying changes between devices so both show the same data.
- **Conflict** — two devices changed the same fact differently while offline.
- **Three-way merge** — comparing your version, their version, and the common
  starting point, to work out what to keep.
- **Base revision** — the last version both devices agreed on.
- **Revision** — a counter that increases each time something changes.
- **Cursor** — your record of how far you have synced. Tells the server what you
  are missing.
- **Idempotent** — sending the same change twice must not cause a problem.
- **`opId`** — a unique id for one operation, generated when queued. Lets the
  server recognise a repeat.
- **Tombstone** — a "this was deleted" marker, so deletions propagate.
- **Last-write-wins** — a simple rule where the newest change always wins. Simple,
  but it silently destroys correct data when clocks disagree.
- **Offline edit** — a change made with no network, synced later.

---

## Understand this first

### Granularity is the conflict-resolution strategy

The instinct is to sync "the Profile" as one document. Do not do that.

```
  Phone                          Laptop
  ─────                          ──────
  edits "email"                  edits "phone"
        │                              │
        └────── sync ──────────────────┘
                     │
        ONE document. Two writers. CONFLICT.
        LWW picks one. The other edit is GONE.
```

Now make each fact its own document:

```
  docId: "email"    ← written by phone
  docId: "phone"    ← written by laptop
  docId: "address"  ← written by neither
                     │
        NO CONFLICT. Both changes land.
```

**Small documents make conflicts disappear by construction rather than by clever merging.** And
in this product the natural unit is already tiny: a `ProfileFact` is one key, one value, one
provenance. You are not inventing a schema to make sync easy — you are noticing that the data
model was already right.

| Document | Granularity | Conflict frequency |
|---|---|---|
| `fact` | One key (`email`, `cgpa`) | Rare — two devices rarely set *the same fact* |
| `verse` | One saved answer | Very rare — a text block |
| `document` | One uploaded file | Never — documents are immutable once stored |
| `setlist` | One form entry | Rare |

**This is the single most important design decision in the backend, and it is a schema decision
made for sync reasons.** Make `Profile` a collection of fact documents rather than one blob, and
90% of conflict-resolution code never gets written.

### What a conflict now actually means

After granularity, the remaining conflict is precise and worth naming:

> Two devices set **the same fact key** to **different values**, while both were offline.

That is rare. And when it happens, **it is not a machine problem — it is a user decision.** Two
people's laptops did not disagree about `cgpa`; the same person entered `8.7` in September and
`8.9` in October and neither device saw the other. Only the user knows which is right.

So the correct behaviour is not a merge algorithm. It is: **keep both, show both, ask.**

### The revision counter is the classic sync bug

Every write needs a monotonic, gap-free-enough, per-user sequence number. Two implementations:

```ts
// ❌ WRONG. Two concurrent writes both read 41, both write 42.
//    One change is now invisible to a cursor-based pull.
const next = (await CountersModel.findById(userId)).value + 1
await CountersModel.updateOne({ _id: userId }, { $set: { value: next } })
await SyncMetaModel.create({ ..., revision: next })
```

```ts
// ✅ Atomic increment-and-read. findOneAndUpdate is a single
//    document operation; MongoDB guarantees atomicity per document.
const counter = await CountersModel.findOneAndUpdate(
  { _id: userId },
  { $inc: { value: 1 } },
  { new: true, upsert: true },
)
const revision = counter.value    // guaranteed unique for this user
```

> **This is the bug that produces "sync randomly misses a change, once every few days."** It is
> not a rare race — two devices syncing at the same moment hit it reliably. And it is
> impossible to reproduce on demand, which makes it a nightmare. `findOneAndUpdate` with
> `$inc` is atomic per document. Use it. There is no reason to do this any other way.
>
> **Gaps are fine. Duplicates are not.** If revision 42 was assigned and the write failed, you
> have a gap. A cursor-based pull asks for `revision > cursor`, so gaps are invisible. What
> breaks you is two documents with the *same* revision, because a cursor cannot order them.

---

## Step 1 — The protocol

Four endpoints. That is the whole protocol.

```
PUSH   POST /sync/push    local changes → server     assigns revisions
PULL   GET  /sync/pull?since=<rev>    → metadata   what changed
FETCH  POST /sync/blobs  docIds       → ciphertext  the actual bytes
CHANGE POST /sync/change-passphrase  → re-wrapped DEKs
```

**`pull` returns metadata only, never ciphertext.** Metadata is small, hot, and content-free —
Chapter 5's `syncmeta` design paying off exactly as intended. The client then asks for the
specific blobs it does not have. A device with nothing new does a `pull` of a few hundred bytes
and stops.

```
  Device A pulls since=40
  syncmeta where revision > 40  →  [{docId:"email", rev:41, device:"A"},
                                   {docId:"phone", rev:42, device:"B"}]
  already has "phone"@42       → skip
  fetch blob "email"           → 1 request, 1 ciphertext
```

---

## Step 2 — `POST /sync/push`

```ts
// apps/api/src/routes/sync.ts
const PushSchema = z.object({
  syncVersion: z.number().int(),
  changes: z.array(z.object({
    opId:       z.string().uuid(),          // client-generated, for idempotency
    docId:      z.string().min(1).max(128),
    docType:    z.enum(["fact", "document", "verse", "setlist"]),
    wrappedDek: z.object({ ciphertext: z.instanceof(Uint8Array), iv: z.instanceof(Uint8Array) }),
    ciphertext: z.instanceof(Uint8Array),
    iv:         z.instanceof(Uint8Array),
    algo:       z.literal("AES-256-GCM"),
    kdfVersion: z.number().int(),
    byteLength: z.number().int().min(0).max(10 * 1024 * 1024),
    /** Base revision this edit was made against. 0 = new document. */
    baseRev:    z.number().int().min(0),
  })).min(1).max(500),
})

export async function push(c: Context) {
  const { syncVersion, changes } = c.req.valid()
  const userId = c.get("userId")!
  const deviceId = c.get("deviceId")!

  if (syncVersion !== SYNC_VERSION) {
    // The client's protocol is older or newer. Tell it to start over
    // rather than corrupt the vault.
    throw new AppError("conflict", "This client needs a full re-sync.", 409, {
      requiredSyncVersion: SYNC_VERSION,
    })
  }

  const applied: Array<{ docId: string; revision: number; status: "applied" | "duplicate" | "conflict" }> = []

  for (const ch of changes) {
    // ── IDEMPOTENCY ──
    // A retried push (mobile network dropped the response) must not
    // apply twice. opId is client-generated and stable across retries.
    const seen = await OpModel.findOne({ userId, opId: ch.opId }).lean()
    if (seen) { applied.push({ ...seen, status: "duplicate" }); continue }

    const existing = await SyncMetaModel.findOne({ userId, docId: ch.docId }).lean()

    // ── CONFLICT DETECTION ──
    // Different device wrote this doc at a revision newer than what
    // we based our edit on. We cannot merge ciphertext.
    if (existing && existing.deviceId?.toString() !== deviceId && existing.revision > ch.baseRev) {
      await OpModel.create({ userId, opId: ch.opId, docId: ch.docId, result: "conflict" })
      applied.push({ docId: ch.docId, revision: existing.revision, status: "conflict" })
      continue     // ← do NOT overwrite. Let the client merge.
    }

    const counter = await CountersModel.findOneAndUpdate(
      { _id: userId }, { $inc: { value: 1 } }, { new: true, upsert: true },
    )

    await BlobModel.findOneAndUpdate(
      { userId, docType: ch.docType, docId: ch.docId },
      { $set: {
          wrappedDek: ch.wrappedDek, ciphertext: ch.ciphertext, iv: ch.iv,
          algo: ch.algo, kdfVersion: ch.kdfVersion, byteLength: ch.byteLength,
        },
        $setOnInsert: { createdAt: new Date() } },
      { upsert: true },
    )
    await SyncMetaModel.findOneAndUpdate(
      { userId, docId: ch.docId },
      { $set: { revision: counter.value, deviceId, updatedAt: new Date(), deleted: false } },
      { upsert: true },
    )
    await OpModel.create({ userId, opId: ch.opId, docId: ch.docId, result: "applied", revision: counter.value })

    applied.push({ docId: ch.docId, revision: counter.value, status: "applied" })
  }

  return c.json({ cursor: await currentCursor(userId), applied })
}
```

### Five decisions in that handler

**`opId` makes push idempotent.** A phone on a train uploads, the request reaches the server, the
response is lost to a dead cell tower, the client retries — and without `opId` you have two
writes, two revisions, and a ghost document. **`opId` is generated once when the change is
*queued*, not when it is *sent*.** That is what makes it stable across retries.

**The conflict path does not overwrite.** It records `conflict` and returns. The server cannot
merge ciphertext — it has no idea what is inside — so its only correct action is to decline and
tell the client. Everything else happens client-side.

**`baseRev` travels with the change.** The client sends the revision it based its edit on. That
one number is how the server detects that someone else wrote in between. Without it, last-write-
wins silently destroys data and no client can tell.

**`existing.deviceId !== deviceId` — same-device writes are not conflicts.** Your own laptop
editing a fact twice is not a conflict. Only *cross-device* writes with a stale base are.

**`SyncMetaModel` upsert on every push.** One document per user per docId, updated in place. The
`revision` always reflects the latest write, so a cursor pull never returns a stale version.

---

## Step 3 — `GET /sync/pull`

```ts
export async function pull(c: Context) {
  const userId = c.get("userId")!
  const since = Number(c.req.query("since") ?? 0)
  const limit = Math.min(Number(c.req.query("limit") ?? 500), 500)

  // Metadata only. No ciphertext on this path, ever.
  const entries = await SyncMetaModel.find({ userId, revision: { $gt: since } })
    .sort({ revision: -1 })
    .limit(limit)
    .select("docId docType revision deviceId updatedAt deleted")
    .lean()

  const cursor = entries.length ? entries[0]!.revision : since

  return c.json({
    entries,
    cursor,
    // Hints let the client skip work it does not need to do.
    hasMore: entries.length === limit,
    serverTime: Date.now(),
  })
}
```

**Why metadata-only is the most important decision on this endpoint.** A user with 200 fact
documents pushes and pulls on every panel open. If `pull` returned ciphertext, that is 200
round trips of megabytes every time you switch tabs. Metadata-only makes the common case — "no
changes" — a sub-kilobyte response, and the expensive path happens only when something actually
changed.

**Descending sort with `limit` for pagination.** Ascending would page oldest-first, so a user
with 10,000 changes pulls 20 round trips before seeing anything new. Descending surfaces the most
recent changes immediately, which is what the user is waiting for.

---

## Step 4 — Client-side merge

The server declined. This runs on the device, where the plaintext exists.

```ts
// packages/vault/src/sync/merge.ts
import type { ProfileFact } from "@refrain/fields"

export type MergeOutcome<T> =
  | { action: "keep-local";  fact: T }
  | { action: "take-remote"; fact: T }
  | { action: "merged";      fact: T; note: string }
  | { action: "conflict";    local: T; remote: T }

/**
 * Three-way merge for a single fact.
 *
 * `base` is the common ancestor — the version BOTH devices started
 * from. Without it, this is last-write-wins with extra steps.
 */
export function mergeFact(
  key: string,
  base: ProfileFact | undefined,
  local: ProfileFact,
  remote: ProfileFact,
): MergeOutcome<ProfileFact> {
  // Only one side changed.
  if (!base) {
    // Both created it. Different values, no common ancestor.
    if (local.value === remote.value) return { action: "keep-local", fact: local }
    return { action: "conflict", local, remote }
  }
  if (base.value === local.value && base.value !== remote.value) {
    return { action: "take-remote", fact: remote }      // only remote changed
  }
  if (base.value !== local.value && base.value === remote.value) {
    return { action: "keep-local", fact: local }        // only local changed
  }
  if (local.value === remote.value) {
    return { action: "merged", fact: local, note: "same change on both devices" }
  }

  // ── Both changed, differently. This is a user decision. ──
  return { action: "conflict", local, remote }
}
```

### Why a true three-way merge and not LWW

```ts
// ❌ Last-write-wins, no base comparison:
return local.updatedAt > remote.updatedAt ? local : remote
```

Consider: your laptop syncs `cgpa: 8.7`. On your phone, months later, you update it to `8.9` and
sync. Then you open the laptop, which has been asleep, and it wakes and pushes `8.7` — a change
it made *before* the phone's. LWW picks the laptop's value because its clock is newer, and your
`8.9` is gone. You now have a stale GPA in your vault and you do not know it.

**With a base comparison**, the laptop's `8.7` has `base.value === "8.9"` — no wait, the laptop
hasn't seen `8.9`, so its base is also `8.7`, meaning "only remote changed" and the merge takes
`8.9`. Correct.

> **Clock skew is not a hypothetical on phones.** Android and iOS routinely have clocks minutes
> apart, and a user who manually set their date wrong can be hours off. Any sync design that
> trusts `Date.now()` for correctness is wrong on your user's actual device. **Revisions from
> the server are the only trustworthy ordering.** Timestamps are for display.

### Preserving conflicts instead of picking

```ts
export interface ConflictRecord {
  factKey: string
  localValue: string
  remoteValue: string
  localAt: number
  remoteAt: number
  detectedAt: number
}

export function applyMerge<T>(
  result: MergeOutcome<T>,
  conflicts: ConflictRecord[],
  key: string,
): T {
  if (result.action === "conflict") {
    conflicts.push({
      factKey: key,
      localValue: (result.local as ProfileFact).value,
      remoteValue: (result.remote as ProfileFact).value,
      localAt: (result.local as ProfileFact).updatedAt,
      remoteAt: (result.remote as ProfileFact).updatedAt,
      detectedAt: Date.now(),
    })
    // Keep local for now so the app works. Surface the conflict.
    return result.local
  }
  return result.fact
}
```

> **Never silently drop the losing value.** Keep it, show it, let the user pick. A vault that
> quietly discards a value is a vault the user cannot trust, and the moment they discover it
> they have to re-verify everything by hand.

```tsx
// The resolution UI — deliberately plain
function ConflictBanner({ conflicts, onResolve }: Props) {
  if (!conflicts.length) return null
  return (
    <div className="rounded-card border border-prov-input p-4">
      <h3 className="font-medium text-ink">
        {conflicts.length} field{conflicts.length > 1 ? "s" : ""} changed on two devices
      </h3>
      <p className="mt-1 text-sm text-ink-muted">
        These were edited while offline somewhere. Pick the one you want to keep.
      </p>
      {conflicts.map((c) => (
        <div key={c.factKey} className="mt-3 flex items-center gap-3">
          <span className="w-24 text-xs text-ink-muted">{c.factKey}</span>
          <button onClick={() => onResolve(c.factKey, c.localValue)}
            className="flex-1 rounded-card border px-3 py-2 text-sm text-ink">
            {c.localValue}
            <span className="block text-xs text-ink-muted">
              This device · {formatDate(c.localAt)}
            </span>
          </button>
          <button onClick={() => onResolve(c.factKey, c.remoteValue)}
            className="flex-1 rounded-card border px-3 py-2 text-sm text-ink">
            {c.remoteValue}
            <span className="block text-xs text-ink-muted">
              Other device · {formatDate(c.remoteAt)}
            </span>
          </button>
        </div>
      ))}
    </div>
  )
}
```

**Show both values with both dates and no recommendation.** Two `cgpa` values, one from each
device, and the user decides in two seconds. Any "we picked for you" is a guess about their life.

---

## Step 5 — The offline queue

```ts
// packages/vault/src/sync/queue.ts
import { db } from "@refrain/vault"

export interface QueuedChange {
  opId: string
  docId: string
  docType: "fact" | "document" | "verse" | "setlist"
  baseRev: number
  payload: SealedPayload
  queuedAt: number
  attempts: number
}

/**
 * opId is minted HERE, at queue time — not at send time.
 * That is what makes a retry idempotent instead of duplicating.
 */
export async function enqueue(change: Omit<QueuedChange, "opId" | "queuedAt" | "attempts">) {
  await db.queue.add({
    ...change,
    opId: crypto.randomUUID(),
    queuedAt: Date.now(),
    attempts: 0,
  })
  notifySyncBadge()          // the little dot in the sidebar
}

export async function flush(limit = 100): Promise<{ sent: number; failed: number }> {
  const batch = await db.queue.orderBy("queuedAt").limit(limit).toArray()
  if (!batch.length) return { sent: 0, failed: 0 }

  let sent = 0, failed = 0

  for (const change of batch) {
    try {
      await api.push([change])

      await db.queue.delete(change.opId)
      // Also bump the local baseRev so we do not re-detect our own write as a conflict.
      await bumpLocalRev(change.docId)
      sent++
    } catch (err) {
      if (isRetryable(err)) {
        change.attempts++
        await db.queue.put(change)          // keep it, try later
      } else {
        await db.queue.delete(change.opId)  // permanent failure, drop it
        await reportPermanent(change, err)
      }
      failed++
      break                                // ordered queue: stop at the first failure
    }
  }
  return { sent, failed }
}

const isRetryable = (err: unknown) =>
  err instanceof TypeError ||                              // network, offline
  (err as { status?: number }).status === 429 ||
  (err as { status?: number })status === 503
```

**Three decisions:**

**Stop at the first failure.** The queue is ordered. If change 3 fails, sending 4–100 creates
out-of-order state on the server, and you have to detect and fix that later. **A sequential queue
that stalls is far better than a parallel one that arrives scrambled.**

**Retryable vs permanent.** `TypeError` is a network failure — retry. A `422` validation error
will never succeed, so retrying it forever wedges the queue permanently and the user's changes
silently stop syncing. **Classify, or your sync silently dies and you never find out.**

**Bump the local revision after a successful push.** Skip this and the next pull sees the server's
new revision as "not yours," the merge compares against a stale base, and your own write comes
back as a conflict against itself. This produces phantom conflicts on every single sync.

---

## Step 6 — When to sync

```ts
// packages/vault/src/sync/scheduler.ts

// The important triggers are the ones nobody remembers to write.
document.addEventListener("visibilitychange", () => {
  if (document.visibilityState === "visible") void sync("visible")
})
window.addEventListener("online", () => void sync("online"))
window.addEventListener("focus", () => void sync("focus"))
window.addEventListener("pagehide", () => {
  // Best-effort. Do not await a network call here; it will be killed.
  void flush().catch(() => {})
})
```

| Trigger | Why |
|---|---|
| App / panel opens | The obvious one |
| **After every fill** | The moment data changes most — a user just filled a form |
| **`visibilitychange` → visible** | The user switched back from a form. Highest-value sync moment there is |
| **`online`** | A phone leaving a tunnel should sync without the user noticing |
| **`focus`** | Catches the alt-tab-and-forget case |
| Interval (5 min) | Backstop only. **Never the primary trigger** |
| **`pagehide`** | Best-effort flush. Will usually be killed. That is fine |

> **Interval-only sync is the classic mistake.** Polling every 30 seconds means the user fills
> a form, closes the tab, and the value is not on their phone for another 25 seconds — or never,
> if they closed the laptop. **Event-driven sync on `visibilitychange` and `online` is both
> cheaper and more correct**, because those are the moments where a change is likely to matter.

```ts
export async function sync(trigger: string) {
  if (!isUnlocked()) return          // never sync while locked
  if (!navigator.onLine) return

  // Serialise. Concurrent syncs race the cursor and duplicate work.
  if (inFlight) return inFlight
  inFlight = (async () => {
    await flush()
    const { entries, cursor } = await api.pull(localCursor)
    const needed = entries.filter((e) => !hasBlob(e.docId) || e.revision > localRev(e.docId))
    if (!needed.length) { setCursor(cursor); return }

    const blobs = await api.fetchBlobs(needed.map((e) => e.docId))
    await mergeIncoming(blobs)          // ← the merge from Step 4
    setCursor(cursor)
    setLastSync(Date.now())
  })().finally(() => { inFlight = null })

  return inFlight
}
```

---

## Step 7 — Test it properly

**Two "devices" in one test run** is the only honest way to test sync. Anything less tests the
happy path.

```ts
// packages/vault/src/sync/__tests__/convergence.test.ts

/** Two independent local stores. Each is a "device". */
function makeDevice(name: string) {
  const blobs = new Map<string, ProfileFact>()
  let cursor = 0
  return {
    name, blobs,
    get cursor() { return cursor },
    set cursor(v: number) { cursor = v },
    get: (id: string) => blobs.get(id),
    put: (id: string, f: ProfileFact) => blobs.set(id, f),
  }
}

describe("two-device convergence", () => {
  let phone: ReturnType<typeof makeDevice>
  let laptop: ReturnType<typeof makeDevice>

  beforeEach(() => { phone = makeDevice("phone"); laptop = makeDevice("laptop") })

  it("a write on one device reaches the other", async () => {
    phone.put("email", fact("r@example.com"))
    await syncAll(phone, laptop)
    expect(laptop.get("email")?.value).toBe("r@example.com")
  })

  it("two devices editing DIFFERENT facts both survive", async () => {
    // ← This is the test that validates the granularity decision.
    phone.put("email", fact("r@example.com"))
    laptop.put("phone", fact("+919876543210"))

    await syncAll(phone, laptop)
    await syncAll(laptop, phone)

    expect(phone.get("phone")?.value).toBe("+919876543210")
    expect(laptop.get("email")?.value).toBe("r@example.com")
  })

  it("two devices editing the SAME fact surfaces a conflict and keeps both", async () => {
    phone.put("cgpa", fact("8.9"))
    laptop.put("cgpa", fact("8.7"))

    await syncAll(phone, laptop)
    await syncAll(laptop, phone)

    // Neither value was silently destroyed.
    expect(conflictsFor("cgpa")).toHaveLength(1)
    expect(conflictsFor("cgpa")[0]).toMatchObject({
      localValue: expect.any(String), remoteValue: expect.any(String),
    })
  })

  it("a stale offline write does not clobber a newer remote write", async () => {
    // ← The clock-skew bug, as a test.
    laptop.put("cgpa", fact("8.7"))
    await syncAll(laptop, server)

    phone.put("cgpa", fact("8.9"))            // newer, correct value
    await syncAll(phone, server)

    await syncAll(laptop, server)             // laptop wakes, pushes its stale 8.7
    await syncAll(laptop, phone)
    await syncAll(phone, laptop)

    // 8.9 survives. LWW would have produced 8.7.
    expect(phone.get("cgpa")?.value).toBe("8.9")
  })

  it("a retried push is idempotent", async () => {
    phone.put("email", fact("r@example.com"))
    await phone.push()                        // succeeds, response lost
    await phone.push()                        // retry with the same opId
    expect(server.docCount()).toBe(1)
    expect(server.maxRevision()).toBe(1)
  })

  it("a delete on one device propagates and does not resurrect", async () => {
    phone.put("pan", fact("ABCDE1234F"))
    await syncAll(phone, laptop)
    await phone.delete("pan")
    await syncAll(phone, laptop)

    expect(laptop.get("pan")).toBeUndefined()
    await syncAll(laptop, phone)               // laptop was offline during the delete
    expect(phone.get("pan")).toBeUndefined()  // ← the resurrect bug
  })
})
```

> **The "stale offline write" test is the one that fails most often on a naive implementation**
> and it is the one that produces silent data corruption in production. It is also the reason
> three-way merge beats LWW, expressed as a test you can read in ten seconds.

> **The delete test is the second-most-common failure.** Deletion propagating without tombstones
> is exactly the resurrect bug from Chapter 5. If this test fails, check `deleted: true` is in
> `syncmeta` and the client honours it.

---

## Step 8 — Sync version and migrations

```ts
export const SYNC_VERSION = 1        // packages/vault/src/sync/version.ts

// Server side
if (syncVersion !== SYNC_VERSION) {
  throw new AppError("conflict", "This client needs a full re-sync.", 409, {
    requiredSyncVersion: SYNC_VERSION,
  })
}
```

```ts
// Client side
async function handleVersionMismatch(required: number) {
  if (required <= SYNC_VERSION) return

  // Safer to start over than to half-migrate.
  await db.blobs.clear()
  await db.facts.clear()
  await db.queue.clear()
  localCursor = 0
  await fullResync()
}
```

**Bump `SYNC_VERSION` for anything structural**: a changed document shape, a new crypto version,
a new merge algorithm, a changed tombstone window.

**Do not bump it for additive changes.** A new optional field does not need a full re-pull, and
forcing every user through a full re-sync to add one field is a genuinely hostile thing to do to
someone on a metered mobile connection.

---

## Step 9 — Commit

```bash
git add -A
git commit -m "feat(sync): revision cursor, idempotent push, client-side 3-way merge, tombstones"
```

Add two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | Facts sync as individual documents, not one Profile blob | Small documents make conflicts disappear by construction. Two devices editing `email` and `phone` must not conflict. |
| 2026-10-XX | Three-way merge with a base, not last-write-wins | Phone clocks drift. Any design trusting `Date.now()` for correctness will overwrite a correct value with a stale one. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`findOneAndUpdate` + `$inc` and why not read-then-write** | **You.** This is the sync bug everyone hits |
| **`mergeFact` and the three branches** | **You** |
| **The granularity decision and its justification** | **You** |
| `baseRev` and conflict detection on the server | **You** |
| `opId` minted at queue time, not send time | **You** |
| The conflict-resolution UI copy | **You** |
| Sync trigger list and the reasoning per trigger | **You** |
| Push/pull handler boilerplate | **OpenCode** |
| `flush` and the retryable classification | **OpenCode**, then add the permanent-failure test yourself |
| The two-device test harness | **OpenCode** |
| The remaining test cases | **OpenCode**, then **predict each result before running it** |

> **Predict-before-run is the practice that makes this chapter teach you anything.** For the
> clock-skew test, write down what you believe will happen and why. If you are right, you
> understand sync. If you are wrong, you have found a real bug before your users do.

---

## Gotchas in this chapter

**"Sync randomly misses a change."** The revision counter race. `findOneAndUpdate` with `$inc`,
always. If you did read-then-write, this is your bug and it will never reproduce on demand.

**Every pull downloads everything.** You returned ciphertext from `pull` instead of metadata.
The `blobs`/`syncmeta` split from Chapter 5 exists precisely so this does not happen.

**Conflicts appear on every sync, about your own writes.** You did not bump `localRev` after a
successful push, so every write comes back as a conflict against itself.

**One failed change blocks the whole queue forever.** You retried a `422` forever instead of
classifying it as permanent.

**Deleted documents resurrect.** No tombstones, or the client ignores `deleted: true`.

**A stale offline write clobbers a newer value.** No `baseRev`, or LWW with `updatedAt`. Run the
clock-skew test; it will fail and tell you exactly why.

**Two tabs log each other out on every sync.** Missing `#inflight` in `SyncSession`.

**Pull takes 20 round trips before showing anything new.** Ascending sort. Sort descending.

**The user pays for data they did not change to sync.** Pull is not metadata-only, or you are
not filtering by `localRev`.

**Sync runs while the vault is locked.** The `isUnlocked()` guard is missing and you are pushing
from a cleared in-memory store.

---

## Verify before moving on

- [ ] Revisions are unique per user under 50 concurrent writes (stress test)
- [ ] `pull` returns metadata only — inspect a response and confirm no ciphertext
- [ ] An unchanged device's `pull` is under 1KB
- [ ] A retried push with the same `opId` applies once
- [ ] Two devices editing different facts both keep their change
- [ ] Two devices editing the same fact produces a conflict with **both** values preserved
- [ ] A stale offline write does not overwrite a newer remote value
- [ ] A delete propagates and does not resurrect after a third sync
- [ ] `visibilitychange`, `online`, and `focus` all trigger a sync
- [ ] Sync is a no-op while locked or offline
- [ ] Concurrent `sync()` calls do not race the cursor
- [ ] A `SYNC_VERSION` mismatch triggers a clean re-sync, not a partial migration
- [ ] Every sync-related log line has been checked against `NEVER_LOG`

---

## Check yourself before Chapter 9

1. **Why does making each fact its own document remove 90% of conflicts?**
2. **What breaks if the revision counter is read-then-write instead of atomic `$inc`?**
3. **Why does the server decline to merge instead of trying?**
4. **What is a `base` in a three-way merge, and what happens without it?**
5. **Why must `opId` be minted when a change is queued rather than when it is sent?**
6. **Why stop the flush at the first failure instead of continuing?**
7. **Why must the local revision be bumped after a successful push?**
8. **Why is `pull` metadata-only?**
9. **Why must the client never sync while locked?**
10. **Why does a retryable/permanent split matter for a sync that looks healthy?**

---

**Next: [Chapter 9 — The API Reference](./07-api-reference.md)** — every endpoint, every
request and response shape, every status code, and the error catalogue. The lookup document.