# Chapter 16 — The Setlist & Verses

> **Day 16 · Goal: the tracker persists, Encore re-fills a stale form, and Verses adapt to length.**
>
> Two features that look like CRUD and are not. The Setlist is why you get a second session;
> Verses are the one thing no competitor in §12's table has at all.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **Setlist** — your list of every form you have prepared. Like a set list at a
  concert.
- **Encore** — re-filling a form you already prepared, using the values as they
  were last time.
- **Verse** — a reusable written answer, such as a motivation essay.
- **Adapt** — shortening a verse to fit a character limit.
- **Stale** — a form whose values are old enough that they might be wrong.
- **Character limit** — the maximum number of characters a form accepts.
- **Compression** — making text shorter while keeping its meaning.
- **Live query** — a database query that updates the screen automatically when
  data changes.
- **Export** — saving everything as a file on your own machine.
- **Delete vault** — permanently removing everything.

---

## Understand this first

### The Setlist is the retention mechanism, not a history page

The obvious fear: *"if Refrain fills the form, the user never comes back."* That fear is wrong
for a structural reason.

**The form gets resubmitted.** Every semester every scholarship. Every new internship
application. Every job change updates your address, your CGPA, your graduation year. **A form
filled in March is stale by September.**

So the Setlist is not a log. It is **a running inventory of forms that will need you again**,
and it is the only surface that gives Refrain a reason to exist between the moment a user needs
it and the moment they need it next.

This is also why **Encore** — re-filling a form you already prepared — *is* the retention loop.
The second fill takes ten seconds. Ten seconds is a habit. An app you love that you do not need
is an app you uninstall.

### Refrain must not observe that you submitted

| Status | Meaning | Who sets it |
|---|---|---|
| `noted` | Seen and saved. Nothing filled. | Refrain, on scan |
| `ready` | Every fillable field mapped | Refrain, on resolve |
| `filled` | Values written to the page | Refrain, on write |
| `submitted` | The user says they submitted | **The user** |
| `skipped` | Blocked or unsupported | Refrain |

> **`submitted` is set by a user tap. Refrain does not observe it.** It *could* — attach a
> submit listener, watch for navigation. **Do not.** Watching for submission means running on
> every page and inferring intent from navigation events, which is surveillance-adjacent and
> lands you in exactly the category §11 exists to avoid. One extra tap, deliberately asked for,
> is both more honest and more defensible. If a reviewer asks "how do you know a form was
> submitted?", the answer is a button — which is a good answer.

### Snapshot the label, not the id — or Encore breaks in six months

A portal reorganises its form between March and September. Field ids move.

```ts
// ❌ Writes to fields that no longer exist. Silently fills nothing.
snapshot: Array<{ id: string; value: string }>

// ✅ Re-resolves by label and degrades honestly
snapshot: Array<{ id: string; label: string; value: string }>
```

**The label is the stable thing.** Human-readable question text changes rarely; DOM ids change
on every redesign. Store both — `id` for the happy path, `label` for the fallback — and Encore
becomes resilient to a portal shipping a redesign.

### Verses are adaptive, not just reusable

§7 Pillar 3 is the most-underestimated feature in the product. A competitor's "saved answers"
feature is a textarea and a dropdown: *pick one of your four saved answers for "About you."*

**That is wrong and it is the gap.** The real problem is not *repetition* — it is **rephrasing**
(§2). The same truth written for a 200-character box must be rewritten for a 2,000-character
one, and for a third-person institutional form, and for a job application that wants motivation
rather than biography.

A Verse is **a long-form answer plus a rule for how to compress it.** Not four saved variants —
that just moves the rephrasing work to the user, which is the thing they were avoiding.

---

## Step 1 — The data model

```ts
// packages/vault/src/setlist.ts
import { db } from "./db/schema"

export type SetlistStatus = "noted" | "ready" | "filled" | "submitted" | "skipped"

export interface SetlistEntry {
  id: string
  url: string
  title: string
  portal: string                 // derived: "Google Forms" | "Naukri" | "Custom"
  status: SetlistStatus
  fieldCount: number
  readyCount: number
  snapshot: Array<{ id: string; label: string; value: string }>
  blocked: boolean
  blockedReason?: string
  firstSeenAt: number
  lastFilledAt?: number
  submittedAt?: number
  /** Edits the user made after we filled. The Phase 1 success metric. */
  corrections?: number
}

/** Same form, different query string = same entry. */
const keyFor = (url: string) => url.split("#")[0]!.split("?")[0]!

export async function noteForm(e: Omit<SetlistEntry, "id" | "firstSeenAt">) {
  const id = keyFor(e.url)
  const existing = await db.setlist.get(id)
  await db.setlist.put({
    ...existing,
    ...e,
    id,
    firstSeenAt: existing?.firstSeenAt ?? Date.now(),
  })
  return id
}

export async function setStatus(id: string, status: SetlistStatus) {
  const e = await db.setlist.get(id)
  if (!e) return
  await db.setlist.put({
    ...e,
    status,
    ...(status === "filled" ? { lastFilledAt: Date.now() } : {}),
    // ONLY on an explicit user tap. Never observed.
    ...(status === "submitted" ? { submittedAt: Date.now() } : {}),
  })
}

/**
 * Only counts a correction against a field we actually wrote. A user
 * typing into a field we left blank is not our mistake, and counting
 * it would make the metric lie upward forever.
 */
export async function recordCorrection(id: string, fieldId: string) {
  const e = await db.setlist.get(id)
  if (!e) return
  const weWrote = new Set(e.snapshot.map((s) => s.id))
  if (!weWrote.has(fieldId)) return
  await db.setlist.put({ ...e, corrections: (e.corrections ?? 0) + 1 })
}

/** Stale = submitted more than ~4 months ago. Every academic cycle. */
const STALE_AFTER_MS = 120 * 864e5

export const isStale = (e: SetlistEntry) =>
  e.status === "submitted" && !!e.submittedAt && Date.now() - e.submittedAt > STALE_AFTER_MS
```

Add the index to the Dexie `stores` declaration:

```ts
this.version(2).stores({
  facts:     "key, updatedAt, provenance.kind",
  documents: "id, kind, addedAt",
  blobs:     "id, recordId",
  setlist:   "id, status, lastFilledAt, submittedAt, portal",
  verses:    "id, kind, updatedAt",
  meta:      "id",
})
```

> **Bump to `version(2)` and keep `version(1)`.** Dexie migrations are cumulative and each
> `version(n)` declares the schema *at* that version. Dropping `version(1)` means a returning
> user's vault fails to open and you have destroyed their profile. **Never renumber or delete a
> Dexie version.** Add the new one above.

---

## Step 2 — Encore

```ts
// packages/vault/src/encore.ts
import type { FormSchema, MappingResult } from "@refrain/fields"
import { resolveAll } from "@refrain/mapping"
import { readProfile } from "./profile"

/**
 * Re-fill a form we filled before.
 *
 * Strategy, in order:
 *   1. Field id still present  → use the stored value verbatim. Fast, exact.
 *   2. Label still present    → use the stored value. Survives a redesign.
 *   3. Neither                → re-resolve from the current profile. Best effort.
 *   4. Still nothing          → needs-input. Honest.
 */
export async function encore(
  snapshot: SetlistEntry["snapshot"],
  schema: FormSchema,
  profile: Profile,
): Promise<MappingResult[]> {
  const byId    = new Map(snapshot.map((s) => [s.id, s]))
  const byLabel = new Map(snapshot.map((s) => [normalizeLabel(s.label), s]))

  // Lazily computed, only if strategy 1 and 2 both miss.
  let fresh: MappingResult[] | null = null
  const getFresh = () => (fresh ??= resolveAll(schema, profile))

  return schema.fields.map((field) => {
    const direct  = byId.get(field.id)
    const labelled = byLabel.get(normalizeLabel(field.label))
    const hit = direct ?? labelled

    if (hit) {
      return {
        schema: field,
        value: {
          value: hit.value,
          confidence: { score: 0.95, reason: "Your answer from last time" },
          provenance: {
            kind: "verse",
            label: "your previous answer",
            extractedAt: new Date().toISOString(),
          },
          editedByUser: false,
        },
        status: "ready" as const,
      }
    }

    const remapped = getFresh().find((r) => r.schema.id === field.id)
    return remapped ?? { schema: field, value: null, status: "needs-input" as const }
  })
}
```

**The three strategies matter and the reasons are specific:**

**Id-first** is exact and instant, and it is right whenever the portal has not changed.

**Label fallback** is what makes Encore survive a redesign. Portal ids are build artefacts;
labels are content. This one function is why the feature keeps working a year later.

**Re-resolving from the profile** catches the field you did not have in March — you uploaded
your marksheet since. It cannot know *which* value you used last time, so it presents the
current best answer rather than pretending.

**`needs-input` at the end, always.** There is no strategy 5 where you guess. If you cannot
identify the field, you ask.

---

## Step 3 — The Setlist screen

```tsx
// apps/web/src/routes/Setlist.tsx
import { useLiveQuery } from "dexie-react-hooks"
import { db, isStale, setStatus } from "@refrain/vault"

const STATUS_STYLE: Record<string, { word: string; tone: string; shape: string }> = {
  noted:     { word: "saved",             tone: "var(--color-ink-muted)",  shape: "pill" },
  ready:     { word: "ready to fill",     tone: "var(--color-prov-profile)", shape: "pill" },
  filled:    { word: "needs your submit", tone: "var(--color-prov-input)", shape: "square" },
  submitted: { word: "done",              tone: "var(--color-prov-profile)", shape: "pill" },
  skipped:   { word: "out of scope",      tone: "var(--color-ink-muted)",  shape: "dashed" },
}

type Filter = "active" | "done" | "all"

export function Setlist() {
  const [filter, setFilter] = useState<Filter>("active")
  const entries = useLiveQuery(
    () => db.setlist.orderBy("lastFilledAt").reverse().toArray(),
    [],
  ) ?? []

  const stale = entries.filter(isStale)
  const shown = entries.filter((e) =>
    filter === "all" ? true
    : filter === "done" ? e.status === "submitted"
    : e.status !== "submitted",
  )

  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Your setlist</h1>
      <p className="mt-1 text-sm text-ink-muted">
        {entries.length} form{entries.length === 1 ? "" : "s"} prepared. All of it on this device.
      </p>

      {/* ── Encore ── */}
      {stale.length > 0 && (
        <section className="mt-6 rounded-card bg-surface-muted p-4">
          <h2 className="font-medium text-ink">{stale.length} worth a rerun</h2>
          <p className="mt-1 text-sm text-ink-muted">
            Your CGPA, address, or year may have changed since you filled these.
          </p>
          <button className="mt-3 rounded-card bg-brand-500 px-4 py-2 text-sm font-medium text-white">
            Encore — refresh and re-fill
          </button>
        </section>
      )}

      {/* ── Filters ── */}
      <div className="mt-6 flex gap-1" role="tablist">
        {(["active", "done", "all"] as const).map((f) => (
          <button key={f} role="tab" aria-selected={filter === f}
            onClick={() => setFilter(f)}
            className={`rounded-full px-3 py-1 text-sm ${
              filter === f ? "bg-brand-500 text-white" : "text-ink-muted hover:bg-surface-muted"
            }`}>
            {f}
          </button>
        ))}
      </div>

      {/* ── Rows ── */}
      <ul className="mt-4 flex flex-col gap-2">
        {shown.map((e) => {
          const s = STATUS_STYLE[e.status]!
          return (
            <li key={e.id} className="flex items-center gap-4 rounded-card border border-surface-muted p-4">
              <div className="min-w-0 flex-1">
                <p className="truncate font-medium text-ink">{e.title}</p>
                <p className="truncate text-xs text-ink-muted">
                  {e.portal} · {e.readyCount}/{e.fieldCount} fields
                  {e.corrections ? ` · ${e.corrections} corrected` : ""}
                </p>
              </div>

              <span className={`chip chip--${s.shape}`} style={{ "--chip": s.tone } as React.CSSProperties}>
                {s.word}
              </span>

              {e.status === "filled" && (
                // The ONLY way `submitted` is ever set.
                <button onClick={() => setStatus(e.id, "submitted")}
                  className="rounded-card border px-3 py-1.5 text-xs font-medium text-ink">
                  I submitted this
                </button>
              )}
            </li>
          )
        })}
      </ul>
    </div>
  )
}
```

### Four decisions in that component

**`useLiveQuery` reacts to Dexie writes.** No refetch, no store sync, no stale UI. This is the
single best reason to put state in IndexedDB rather than a Zustand store you must rehydrate by
hand. It also means `setStatus` needs no `await` in the component and no invalidation.

**The `corrections` count is visible to the user, quietly.** Not a dashboard, not a score —
just `2 corrected` in the row subtitle. Users who see that number trust you more, not less,
because it admits you were imperfect. Hiding it is the dishonest choice.

**`"I submitted this"` is a button and nothing else.** No auto-detection anywhere near it.

**Filter tabs use `role="tablist"` / `role="tab"` / `aria-selected`.** Real ARIA tabs have arrow-key
navigation. If you add the roles, implement the keyboard; otherwise a screen reader announces a
tab that does not behave like one.

---

## Step 4 — Export and delete

These are **trust features.** A user evaluating a tool that holds their marksheet looks for both
before they look at the features. Their absence reads as "your data is held hostage."

```ts
// packages/vault/src/export.ts
export async function exportVault(key: CryptoKey): Promise<Blob> {
  const [facts, documents, setlist, verses] = await Promise.all([
    db.facts.toArray(),
    db.documents.toArray(),
    db.setlist.toArray(),
    db.verses.toArray(),
  ])

  const manifest = {
    format: "refrain-export",
    version: 1,
    exportedAt: new Date().toISOString(),
    kdf: (await db.meta.get("meta"))!.kdf,
    counts: {
      facts: facts.length, documents: documents.length,
      setlist: setlist.length, verses: verses.length,
    },
  }

  return new Blob(
    [JSON.stringify(manifest, null, 2), JSON.stringify({ facts, documents, setlist, verses })],
    { type: "application/json" },
  )
}

export async function deleteVault() {
  // Order matters. Blobs first so a crash mid-delete leaves
  // orphaned rows, not orphaned megabytes of ciphertext.
  await db.blobs.clear()
  await db.documents.clear()
  await db.facts.clear()
  await db.verses.clear()
  await db.setlist.clear()
  await db.meta.clear()
  vault.lock()
}
```

> **The export manifest includes `kdf` but never the key.** That is deliberate and it is a
> decision you will be asked about: the salt and iteration count let you re-derive the key from
> a passphrase, so the export is genuinely re-importable, but the export file alone cannot
> decrypt anything. **Export is a backup, not a key.**

---

## Step 5 — Verses

```ts
// packages/vault/src/verses.ts
import type { Provenance } from "@refrain/fields"

export type VerseKind = "motivation" | "about" | "summary" | "statement"

export interface Verse {
  id: string
  kind: VerseKind
  /** The full version. Never edited to fit a box. */
  body: string
  /** Extra facts available to draw on when compressing. */
  facts?: string[]
  audience?: string        // "scholarship" | "job" | "research" | "generic"
  updatedAt: number
  provenance: Provenance
}
```

### The adaptation function

```ts
// packages/vault/src/adapt.ts

interface Field {
  label: string
  /** Character budget from the form itself, or Infinity. */
  maxLength?: number
}

/**
 * Compress a Verse to fit a character budget, preserving meaning.
 *
 * Deterministic and testable. No model in the default path — a 200-char
 * compression of a 2,000-char answer is a summarisation problem, and
 * a wrong summary on a motivation letter is worse than a truncation
 * the user can see and fix.
 */
export function adaptVerse(verse: Verse, field: Field): string | null {
  const body = verse.body.trim()
  if (!body) return null

  // No budget stated → the whole verse. Most forms under-specify.
  if (!field.maxLength || field.maxLength >= body.length) return body

  const sentences = body.split(/(?<=[.!?])\s+/).filter(Boolean)

  // 1. Whole sentences that fit, in order. Never a mid-sentence cut.
  let out = ""
  for (const s of sentences) {
    if ((out + " " + s).trim().length > field.maxLength) break
    out = (out + " " + s).trim()
  }
  if (out) return out

  // 2. Nothing fit. Fall back to the longest clause, then truncate
  //    on a word boundary and mark it visibly.
  const longest = [...sentences].sort((a, b) => b.length - a.length)[0] ?? body
  const cut = longest.slice(0, field.maxLength - 1).replace(/\s+\S*$/, "")
  return `${cut}…`
}
```

**Four rules, and each one is a refusal to be clever:**

**Sentence boundaries, never mid-word cuts.** A truncated word on a motivation letter reads as
a bug, and a bug on a motivation letter costs the application.

**Sentence order preserved.** Reordering sentences to pack more in produces nonsense. Readability
beats density.

**No model in the default path.** This is a compression problem, and a wrong compression is
worse than a visible truncation. If a user wants AI adaptation, Chapter 14's local model path is
available — opt-in, off by default.

**`…` marks a truncation.** The user must be able to tell that Refrain shortened something. A
silent truncation is a lie about the length of their answer.

```tsx
// apps/web/src/routes/Verses.tsx — the review is mandatory
export function Verses() {
  const [draft, setDraft] = useState<Verse | null>(null)

  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Verses</h1>
      <p className="mt-1 text-sm text-ink-muted">
        Write it once. Refrain shortens it to fit whatever the form asks for.
      </p>

      {draft && (
        <div className="mt-4 flex items-center gap-3 rounded-card bg-surface-muted p-3 text-sm">
          <span className="text-ink-muted">Preview for a 200-character box:</span>
          <span className="flex-1 text-ink">{adaptVerse(draft, { maxLength: 200 })}</span>
        </div>
      )}

      <textarea
        value={draft?.body ?? ""}
        onChange={(e) => setDraft({ ...draft, body: e.target.value } as Verse)}
        placeholder="Why do you want this?"
        rows={10}
        className="mt-6 w-full rounded-panel border border-surface-muted bg-surface p-4 text-ink"
      />

      <p className="mt-2 text-right text-xs text-ink-muted">
        {draft?.body.length ?? 0} characters
      </p>
    </div>
  )
}
```

> **The live preview is the feature.** Watching your 900-character verse compress into a
> 200-character box, in real time, as you type, teaches the behaviour in two seconds. Without
> it, users write 200 characters, then discover on a form that the box is 2,000 and their
> answer is thin. The preview is the difference between understanding and guessing.

### The mapping engine's verse layer

Add this to Chapter 14's pipeline as **layer 3.5**, between attributes and inference:

```ts
// Layer 3.5 — a Verse for this kind of field, adapted to the box.
function layerVerses(label: string, profile: Profile, field: FormFieldSchema) {
  const kind = verseKindFor(label)      // "motivation" | "about" | "summary" | "statement"
  if (!kind) return null
  const verse = profile.verses?.find((v) => v.kind === kind && audienceMatches(v, label))
  if (!verse) return null

  return {
    key: `verse:${verse.id}`,
    fact: {
      value: adaptVerse(verse, { label, maxLength: field.maxLength })!,
      confidence: 0.9,   // high — the user wrote this themselves
      provenance: {
        kind: "verse" as const,
        label: `your verse: ${verse.kind}`,
        extractedAt: new Date().toISOString(),
      },
      updatedAt: verse.updatedAt,
    },
  }
}
```

**Extract `maxLength` in Chapter 13's content script.** You are already reading the field; add
`maxLength: el.maxLength > 0 ? el.maxLength : undefined` to `FormFieldSchema`. Without it the
adaptation has no budget and the whole feature degrades to "paste the whole thing."

---

## Step 6 — Commit

```bash
git add -A
git commit -m "feat(setlist): tracker, encore 3-strategy re-fill, verse adaptation, export"
```

Add three rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | `submitted` is a user tap, never observed | Observing submission means running on every page and inferring intent from navigation. §11. |
| 2026-10-XX | Encore snapshot stores the label, not just the id | Portal ids are build artefacts that change on every redesign. Labels survive. |
| 2026-10-XX | Verse compression is deterministic, no model | A wrong summary on a motivation letter is worse than a visible truncation. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`encore()` and the three strategies** | **You.** The label fallback is the insight |
| **`adaptVerse` and the sentence-boundary rule** | **You** |
| The `STALE_AFTER_MS` heuristic | **You** |
| `recordCorrection` and why it counts only fields we wrote | **You** |
| Setlist row copy and status wording | **You** |
| `useLiveQuery` wiring and filters | **OpenCode** |
| `exportVault` / `deleteVault` and the clear ordering | **OpenCode** — then verify the order yourself |
| Test suite for `adaptVerse` edge cases | **OpenCode**, then read each and predict it first |
| `exportVault` blob assembly | **OpenCode** |

---

## Gotchas in this chapter

**Encore writes to fields that no longer exist.** You stored only ids. Store the label and
re-resolve — see Step 2.

**A deleted document reappears after sync.** You have no tombstones. §15's `syncmeta` collection
exists precisely for this, and it is not optional once sync ships.

**`useLiveQuery` returns `undefined` first, then content.** That is "not loaded yet," not a bug.
`?? []` renders an empty Setlist and then pops in — a flash of *"you have no forms"* on every
load. Render a skeleton for the `undefined` case.

**The Setlist shows forms you refused.** You call `noteForm` on scan, including blocked pages.
Do not note a form you refused.

**Your Dexie upgrade destroyed everyone's vault.** You renumbered or deleted `version(1)`. See
Step 1 — **always add a new version, never remove an old one.**

**`adaptVerse` returns `null` and the row goes `needs-input`.** Your Verse body is empty, or
`field.maxLength` is undefined and the body is empty after trim. Check the body is non-empty
before the budget check.

**`orderBy("lastFilledAt")` misses entries with no `lastFilledAt`.** They sort to the end, which
is right, but a `noted` entry you never filled is now invisible under "active." Sort by
`firstSeenAt` for never-filled entries, or filter them separately.

**The textarea re-renders the whole list on every keystroke.** At four verses it is invisible. At
forty it drops frames. Debounce the preview, or move it to a child component with its own state.

**Export produces a file you cannot import.** You exported the manifest but not a version
migration path. Add `format` + `version` (you have them) and a migration function before you
need one — retrofitting a format version is painful.

---

## Verify before moving on

- [ ] Setlist persists across reloads, persists across Chrome restarts (after unlock)
- [ ] Every status renders with the correct chip shape
- [ ] `"I submitted this"` is a button and there is **no** submission observer anywhere
- [ ] `grep -rniE "addEventListener.*submit|navigation|onBeforeNavigate" packages/ apps/` returns nothing
- [ ] Encore re-fills a form and uses the stored value
- [ ] Encore survives a simulated portal redesign (change the ids, keep the labels)
- [ ] A stale entry triggers the Encore prompt
- [ ] Export produces a re-importable archive; delete clears everything
- [ ] `adaptVerse` truncates only on sentence boundaries, marks with `…`
- [ ] The verse preview updates live as you type
- [ ] `maxLength` is extracted in the content script and reaches the adapter
- [ ] Correction counts only increment for fields Refrain wrote

---

## Check yourself before Chapter 17

1. **Why is the Setlist a retention mechanism rather than a history log?**
2. **Why does Refrain not observe submission, and what is the alternative?**
3. **Why does the Encore snapshot store the label?**
4. **What happens to Encore on strategy 4, and why is there no strategy 5?**
5. **Why does `adaptVerse` refuse to use a model?**
6. **What is the live preview for, and what does its absence cost?**
7. **Why does `recordCorrection` check that we wrote the field?**
8. **Why must you never delete a Dexie `version(1)`?**

---

**Next: [Chapter 17 — Document Extraction](./17-document-extraction.md)** — pdf.js text-layer
reconstruction, the scan detection that routes to Tesseract, the OCR confidence rule that keeps
photographed marksheets safe, and provenance you can verify in two seconds.