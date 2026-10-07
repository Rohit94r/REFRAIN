# Chapter 9 — Fermata

> **Days 11–13 · Goal: one Google Form filled end-to-end. This is the MVP.**
>
> A fermata is a held note — the performer waits for the conductor. This screen is your §11
> human gate made visible. **It is the product.** Spend your best hours here.

---

## Understand this first

### The double gate — and why there are two stops, not one

`refrain.md` §20 puts the build order as:

```
7. Review screen — read-only render of proposed values with provenance
8. Write values to the page
9. User presses submit          ← MVP reached
```

That ordering is not incidental. **Read the review screen *before* you touch the page.** It
means Refrain never mutates a form you have not first seen and approved. Two stops:

```
   Scan  →  Propose  →  **YOU REVIEW**  →  **YOU APPROVE**  →  Fill  →  **YOU SUBMIT**
              ↑              ↑                     ↑                         ↑
         machine          human                  human                   human
```

If you fill first and review after, every correction becomes a *reversal* — you have already
put a wrong value into a government form, and undoing it requires the field to be repopulated.
Reviewing first means the wrong value **never existed anywhere.** That is a categorically
safer product, and it costs one extra click.

Two clicks you will never get charged for, because the user does not perceive them as clicks —
they perceive it as "it showed me what it was going to do."

### The mascot's Fermata state is not a brand flourish here

It is a **state machine label.** Rest → Listening → Fermata → Attacca maps exactly onto the
flow below, and the user can learn the product in one session because the mascot told them
where they were.

```
   Rest          "Ready when you are"
     │  open panel
     ▼
   Listening     "Heard 14 fields on this page"     ← scanning, machine
     │  resolve
     ▼
   Fermata       "2 blanks, 1 attachment"           ← HELD. Waiting for you.
     │  approve
     ▼
   Attacca       "Filled — review and submit"       ← go on. Your move.
```

**Nothing happens without a human transition.** No timeout, no auto-advance, no "proceeding
in 5 seconds…". A gate that resolves itself is a suggestion.

### Every row carries provenance. Every row. Non-negotiable.

This is the screen's entire reason to exist. Not the typography, not the mascots — the fact
that a 19-year-old can see, for each value, exactly where it came from before it goes anywhere.

So the row has five parts, and all five are load-bearing:

```
┌──────────────────────────────────────────────────────────────────────┐
│  Your full name *                                    ✓ from your…    │
│  ┌────────────────────────────────────────────────────────────────┐  │
│  │ Rohit Jadhav                                              ✎    │  │
│  └────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────────────────────────────────────────┘
   1 label +        2 the value, always editable        3 provenance chip
     required        in one tap                           + confidence
```

### The four row states are a colour AND a shape AND a word

Never colour alone — §9, and the accessibility argument from Chapter 3.

| Status | Value shown | Shape | Colour | The word |
|---|---|---|---|---|
| `ready` | filled | flat pill | brand green | `from your profile` |
| `needs-review` | filled | **square** | amber | `check this` |
| `needs-input` | empty | **dashed** | amber | `you type this` |
| `unsupported` | empty | **dashed** | grey | `do this by hand` |

The word is what a screen reader announces and what a colour-blind user reads. **If a state
has no word, it does not exist.**

---

## Step 1 — The state machine

Your Chapter 3 sketch becomes a real machine now, because the states have guards.

```bash
pnpm add xstate zustand
```

```ts
// apps/extension/entrypoints/sidepanel/machine.ts
import { setup, assign } from "xstate"
import type { FormSchema, MappingResult } from "@refrain/fields"

export interface ReviewContext {
  schema: FormSchema | null
  results: MappingResult[]
  /** Values the user has overridden in this session. Never re-resolved. */
  edits: Record<string, string>
  wrote: boolean
  error: string | null
}

export type ReviewEvent =
  | { type: "SCAN" }
  | { type: "SCANNED"; schema: FormSchema; results: MappingResult[] }
  | { type: "EDIT"; id: string; value: string }
  | { type: "APPROVE" }
  | { type: "WROTE" }
  | { type: "WRITE_FAILED"; error: string }
  | { type: "RETRY" }
  | { type: "LOGGED" }
  | { type: "LEAVE" }

const hasNeedsInput = (c: ReviewContext) =>
  c.results.some((r) => r.status === "needs-input")

const allAnswered = (c: ReviewContext) =>
  c.results.every(
    (r) => r.value !== null || r.schema.kind === "file" || !r.schema.required,
  )

export const reviewMachine = setup({
  types: {
    context: {} as ReviewContext,
    events: {} as ReviewEvent,
  },
  guards: {
    hasNeedsInput,
    allAnswered,
    // The one that matters most. See below.
    captchaPresent: ({ context }) => context.schema?.captchaDetected === true,
    pageBlocked: ({ context }) => context.schema?.blocked === true,
  },
}).createMachine({
  id: "review",
  initial: "idle",
  context: { schema: null, results: [], edits: {}, wrote: false, error: null },

  states: {
    idle: {
      on: { SCAN: { target: "scanning" } },
    },

    scanning: {
      on: {
        SCANNED: {
          target: "reviewing",
          actions: assign(({ event }) => ({
            schema: event.schema,
            results: event.results,
          })),
        },
        LEAVE: { target: "idle" },
      },
    },

    reviewing: {
      entry: "deriveMascotState",
      on: {
        EDIT: {
          actions: assign(({ event, context }) => ({
            edits: { ...context.edits, [event.id]: event.value },
            // An edit is authoritative. Flip editedByUser so the chip says so
            // and the correction metric in Chapter 10 can actually be measured.
            results: context.results.map((r) =>
              r.schema.id === event.id && r.value
                ? { ...r, value: { ...r.value, value: event.value, editedByUser: true } }
                : r,
            ),
          })),
        },
        APPROVE: [
          // §11: never fill a form we have refused to touch.
          { target: "refused", guard: "pageBlocked" },
          // CAPTCHA: hand control back, do not attempt to proceed.
          { target: "awaiting-human", guard: "captchaPresent" },
          // Required field with no answer: refuse to write a partial form.
          { target: "incomplete", guard: ({ context }) =>
              context.results.some(
                (r) => r.schema.required && !r.schema.options?.length &&
                       r.schema.kind !== "file" &&
                       !r.value && !(r.schema.id in (context.edits ?? {})),
              ) },
          { target: "writing" },
        ],
        LEAVE: { target: "idle" },
      },
    },

    /** User must solve the CAPTCHA themselves. Refrain waits. */
    awaiting-human: {
      on: {
        EDIT: { actions: "noop" },
        APPROVE: { target: "writing" },   // they solved it; try again
        LEAVE: { target: "idle" },
      },
    },

    incomplete: {
      on: {
        EDIT: { target: "reviewing", actions: "applyEdit" },
        APPROVE: { target: "reviewing" },
        LEAVE: { target: "idle" },
      },
    },

    refused: { on: { LEAVE: { target: "idle" } } },
    writing: {
      on: {
        WROTE: { target: "filled" },
        WRITE_FAILED: { target: "writeFailed", actions: assign({ error: ({ event }) => event.error }) },
      },
    },
    writeFailed: {
      on: {
        RETRY: { target: "writing" },
        LEAVE: { target: "idle" },
      },
    },

    /**
     * TERMINAL. There is no transition out of `filled`.
     *
     * Refrain has written its values. Refrain does not submit. There is
     * deliberately no SUBMIT event, no auto-click, no timer. If you add
     * one, you have not built this product — you have built the thing
     * §11 is a policy against, and `grep` for it should return nothing.
     */
    filled: {
      type: "final",
    },
  },
})
```

> **`filled` is `type: "final"` with no exit.** That is the whole product in one line. There is
> no `SUBMIT` event to dispatch, so there is no code path that could click the button. The
> human gate is not a convention you follow — it is a state the machine cannot leave.

**Why guards and not `if` statements in the component.** Because guards are visible in one
place, testable without rendering, and impossible to forget. The `pageBlocked` guard means
§11's refusal cannot be bypassed by a future contributor adding a button, because the
transition does not exist.

---

## Step 2 — The store

```ts
// apps/extension/entrypoints/sidepanel/store.ts
import { create } from "zustand"
import type { FormSchema, MappingResult } from "@refrain/fields"
import { vault } from "@refrain/vault"

interface ReviewState {
  schema: FormSchema | null
  results: MappingResult[]
  busy: boolean
  error: string | null
  scan: () => Promise<void>
  approve: (values: Array<{ id: string; value: string }>) => Promise<WriteOutcome>
}

export interface WriteOutcome {
  filled: number
  failed: Array<{ label: string; reason: string }>
}

export const useReview = create<ReviewState>((set, get) => ({
  schema: null,
  results: [],
  busy: false,
  error: null,

  scan: async () => {
    set({ busy: true, error: null })
    try {
      const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })
      if (!tab?.id) throw new Error("No active tab.")

      // Single round trip. If the content script is not injected, say so in
      // a sentence the user can act on.
      const res = await chrome.tabs.sendMessage(tab.id, { type: "REFRAIN/SCHEMA" })
      if (!res?.schema) throw new Error("Reload the page and try again.")

      const { resolveAll } = await import("@refrain/mapping")
      const profile = await vault.readProfile()
      const results = resolveAll(res.schema as FormSchema, profile)

      set({ schema: res.schema as FormSchema, results, busy: false })
    } catch (e) {
      set({ busy: false, error: (e as Error).message })
    }
  },

  approve: async (values) => {
    set({ busy: true, error: null })
    try {
      const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })
      const res = await chrome.tabs.sendMessage(tab!.id!, { type: "REFRAIN/FILL", values })

      // Never report success from the absence of an error. Count the results.
      const outcome: WriteOutcome = {
        filled: res.results.filter((r: { ok: boolean }) => r.ok).length,
        failed: res.results
          .filter((r: { ok: boolean }) => !r.ok)
          .map((r: { id: string; reason?: string }) => ({
            label: get().results.find((x) => x.schema.id === r.id)?.schema.label ?? r.id,
            reason: r.reason ?? "Could not write",
          })),
      }

      set({ busy: false })
      return outcome
    } catch (e) {
      set({ busy: false, error: (e as Error).message })
      return { filled: 0, failed: [] }
    }
  },
}))
```

> **`import()` for `@refrain/mapping`, not a static import.** In a side panel, a static
> import of a package that touches the DOM at module scope will crash the panel on open.
> Dynamic import defers it until you actually scan. If `mapping` ever grows a DOM
> reference, you will find this out at 9am rather than from a user report.

---

## Step 3 — The row

This component is where the product is won or lost.

```tsx
// apps/extension/entrypoints/sidepanel/components/ReviewRow.tsx
import { useState } from "react"
import type { MappingResult } from "@refrain/fields"
import { ProvenanceChip } from "@refrain/ui/provenance"

interface Props {
  result: MappingResult
  onEdit: (id: string, value: string) => void
}

const STATUS_COPY: Record<MappingResult["status"], { word: string; tone: string; shape: string }> = {
  ready:         { word: "from your profile", tone: "prov-profile",  shape: "pill" },
  "needs-review":{ word: "check this",         tone: "prov-input",    shape: "square" },
  "needs-input": { word: "you type this",     tone: "prov-input",    shape: "dashed" },
  unsupported:   { word: "do this by hand",    tone: "ink-muted",     shape: "dashed" },
}

export function ReviewRow({ result, onEdit }: Props) {
  const [editing, setEditing] = useState(false)
  const { schema, value, status } = result
  const copy = STATUS_COPY[status]

  // Never render a status word the user cannot act on.
  if (status === "unsupported") {
    return (
      <li className="row row--unsupported">
        <span className="row__label">{schema.label}</span>
        <p className="row__aside">Refrain will not fill this one. {schema.kind === "file"
          ? "Attachments have to be added by hand."
          : "This field type is not supported yet."}</p>
      </li>
    )
  }

  const isEmpty = value === null
  const showLock = value !== null && value.confidence.score < 0.85

  return (
    <li className={`row row--${status}`}>
      <div className="row__head">
        <label htmlFor={`f-${schema.id}`} className="row__label">
          {schema.label || "Unlabelled field"}
          {schema.required && <span aria-label="required" className="row__req">*</span>}
        </label>

        <div className="row__meta">
          {value && (
            <span className={`chip chip--${copy.shape}`} data-tone={copy.tone}>
              {showLock && <span aria-hidden>🔒</span>}
              {copy.word}
            </span>
          )}
          {value && !showLock && value.provenance && (
            <ProvenanceChip
              provenance={value.provenance}
              locked={false}
              editedByUser={value.editedByUser}
            />
          )}
        </div>
      </div>

      {editing || isEmpty ? (
        <input
          id={`f-${schema.id}`}
          autoFocus={isEmpty}
          list={schema.options?.length ? `opts-${schema.id}` : undefined}
          defaultValue={value?.value ?? ""}
          onBlur={(e) => { onEdit(schema.id, e.target.value); setEditing(false) }}
          onKeyDown={(e) => {
            if (e.key === "Enter") e.currentTarget.blur()
            if (e.key === "Escape") { setEditing(false) }
          }}
          placeholder={isEmpty ? "Type this once" : undefined}
          className="row__input"
          // Announce the label, the value, AND the provenance together.
          // A screen reader user must not have to hunt for the chip.
          aria-describedby={`p-${schema.id}`}
        />
      ) : (
        <button
          id={`f-${schema.id}`}
          onClick={() => setEditing(true)}
          className="row__value"
        >
          {value?.value}
          <span className="row__pencil" aria-hidden>✎</span>
        </button>
      )}

      {/* Datalist gives a suggestion list for free, keyboard accessible, no dependency. */}
      {schema.options?.length ? (
        <datalist id={`opts-${schema.id}`}>
          {schema.options.map((o) => <option key={o} value={o} />)}
        </datalist>
      ) : null}

      <span id={`p-${schema.id}`} className="sr-only">
        {value
          ? `${copy.word}. From ${value.provenance.label}${value.provenance.locator ? `, ${value.provenance.locator}` : ""}.`
          : copy.word}
      </span>
    </li>
  )
}
```

**Six decisions in that component:**

**The value is a `<button>`, not text.** Inline editing is one tap. A user correcting one
value out of fourteen should not have to click an edit icon, wait for a modal, and dismiss it.

**`<datalist>` instead of a combobox library.** Free, keyboard-accessible, zero dependencies,
and it suggests without hiding options. You avoided `select`/`combobox` in Chapter 3 for
exactly this reason — you did not know you needed one yet.

**The provenance is in an `sr-only` span, not only a chip.** A screen reader user must hear
"from marksheet.pdf, page 2" *in the same breath* as the value. Two separate announcements
means the trust information arrives too late to matter.

**`autoFocus` only when empty.** Focusing on load for fourteen rows would be chaos. Focusing
the first empty field is helpful; focusing a value the user did not ask about is hostile.

**Escape reverts, Enter commits.** Both via blur. Standard, expected, invisible.

**`showLock` at 0.85, not at `REVIEW_THRESHOLD`.** Two different lines. `REVIEW_THRESHOLD`
(0.75) is "will it fill." 0.85 is "is it confident enough not to warn you." A value at 0.78
fills and warns. If you use one number for both, every filled row either warns or none do.

---

## Step 4 — The screen

```tsx
// apps/extension/entrypoints/sidepanel/components/Review.tsx
import { useMachine } from "@xstate/react"
import { reviewMachine, type ReviewEvent } from "../machine"
import { useReview } from "../store"
import { Mascot } from "@refrain/ui"
import { FermataFooter } from "./FermataFooter"
import { NeedsAttention } from "./NeedsAttention"

export function Review() {
  const [state, send] = useMachine(reviewMachine)
  const store = useReview()
  const { results } = store

  const ready = results.filter((r) => r.status === "ready").length
  const needs = results.filter((r) => r.status === "needs-input" || r.status === "needs-review").length

  return (
    <div className="flex h-full flex-col bg-surface">
      <header className="border-b border-surface-muted p-4">
        <Mascot
          state={
            state.matches("scanning") ? "listening"
            : state.matches("reviewing") || state.matches("incomplete") ? "fermata"
            : state.matches("filled") ? "attacca"
            : "idle"
          }
          ready={ready}
          needs={needs}
        />
      </header>

      <main className="flex-1 overflow-y-auto px-4 py-3">
        {state.matches("refused") && (
          <Refusal reason={state.context.schema?.blockedReason} />
        )}

        {state.matches("awaiting-human") && (
          <NeedsAttention
            title="This page has a CAPTCHA"
            body="Solve it in the tab, then press Try again. Refrain does not attempt to solve CAPTCHAs, and it will not submit anything while one is present."
          />
        )}

        {(state.matches("reviewing") || state.matches("incomplete")) && (
          <>
            <SummaryBar results={results} />
            <ul className="flex flex-col gap-2">
              {results.map((r) => (
                <ReviewRow key={r.schema.id} result={r}
                  onEdit={(id, value) => send({ type: "EDIT", id, value })} />
              ))}
            </ul>
            <ProvenanceLegend />
          </>
        )}

        {state.matches("filled") && <FilledSummary results={results} />}
      </main>

      <FermataFooter
        state={state}
        needs={needs}
        onApprove={() => send({ type: "APPROVE" })}
        onEdit={(id, v) => send({ type: "EDIT", id, value: v })}
      />
    </div>
  )
}
```

### The refusal screen

```tsx
function Refusal({ reason }: { reason?: string }) {
  return (
    <div className="rounded-panel bg-surface-muted p-6 text-center">
      <h2 className="font-semibold text-ink">Refrain will not fill this page</h2>
      <p className="mt-2 text-sm text-ink-muted">
        {reason ?? "This page is outside what Refrain handles."}
      </p>
      <p className="mt-4 text-xs text-ink-muted">
        Identity verification, payments, and competitive exams are deliberately out of scope.
        It is a short list, and it is not negotiable.
      </p>
    </div>
  )
}
```

> **Do not apologise, and do not offer an override.** There is no "yes, fill it anyway."
> §11's no-list is the product's spine. A soft refusal that the user can click through is a
> refusal that will be clicked through, and then you have built the thing you promised not to
> build. This screen is also your best marketing asset: it is the clearest possible statement
> of what you are.

### The summary bar — one honest sentence

```tsx
function SummaryBar({ results }: { results: MappingResult[] }) {
  const ready  = results.filter((r) => r.status === "ready").length
  const needs  = results.filter((r) => r.status === "needs-input").length
  const review = results.filter((r) => r.status === "needs-review").length
  const skip   = results.filter((r) => r.status === "unsupported").length

  return (
    <p className="mb-3 rounded-card bg-surface-muted px-3 py-2 text-sm text-ink">
      I can fill <strong>{ready}</strong> of {results.length}.
      {needs > 0 && <> You type <strong>{needs}</strong>.</>}
      {review > 0 && <> <strong>{review}</strong> worth a look.</>}
      {skip > 0 && <> <strong>{skip}</strong> by hand.</>}
    </p>
  )
}
```

**"I can fill 12 of 14. You type 2."** One sentence, no jargon, no percentages, no confidence
scores. It sets the expectation before the user reads a single row, which is what makes the
partial result feel like a plan rather than a failure.

---

## Step 5 — Highlighting the page

The review screen is in the panel; the fields are in the tab. **Connect them visually** or the
user is looking at two unrelated places.

```css
/* apps/extension/entrypoints/content/content.css */
.refrain-target {
  outline: 2px solid var(--refrain-brand, oklch(0.68 0.15 62)) !important;
  outline-offset: 1px;
  border-radius: 4px;
  transition: outline-color 150ms ease;
}
.refrain-target--focus {
  outline-width: 3px !important;
  outline-color: oklch(0.62 0.19 25) !important;
}
.refrain-target--skipped {
  outline: 2px dashed oklch(0.52 0.01 60) !important;
}
```

```ts
// in the content script, on APPROVE
for (const r of results) {
  if (r.status === "unsupported") continue
  index.get(r.schema.id)?.classList.add("refrain-target")
  targets.push(r.schema.id)
}

// Focus highlight on row hover — in the PANEL, highlight in the PAGE
// This is the detail that makes the product feel connected.
```

**Wire panel hover → page highlight.** Side panel button:

```tsx
onMouseEnter={() => chrome.tabs.sendMessage(tabId, { type: "REFRAIN/HIGHLIGHT", id })}
onMouseLeave={() => chrome.tabs.sendMessage(tabId, { type: "REFRAIN/HIGHLIGHT", id: null })}
```

> **This one detail is worth an hour.** Row 7 of the review screen outlines field 7 of the
> form as you move your mouse across it. It is the difference between "a panel that shows me
> some data" and "a companion that is looking at the page with me." §9 says the mascot makes
> someone smile once then becomes invisible — **this is the same idea applied to the
> interaction.**

---

## Step 6 — Attacca: after the write

```tsx
function FilledSummary({ results }: { results: MappingResult[] }) {
  const ok = results.filter((r) => r.status === "ready").length
  const failed = results.filter((r) => r.status === "unsupported").length

  return (
    <div className="rounded-panel bg-surface-muted p-5">
      <h2 className="font-semibold text-ink">Filled {ok} fields.</h2>

      <p className="mt-3 text-sm text-ink">
        <strong>Now switch to the tab and check it.</strong> Refrain has not submitted
        anything and never will — that part is yours.
      </p>

      {failed > 0 && (
        <p className="mt-3 text-sm text-ink-muted">
          {failed} field{failed > 1 ? "s" : ""} left for you:{" "}
          {results.filter((r) => r.status === "unsupported")
            .map((r) => r.schema.label).join(", ")}.
        </p>
      )}

      <a
        href="#"
        onClick={(e) => { e.preventDefault(); chrome.tabs.update({ active: true }) }}
        className="mt-4 inline-block rounded-card bg-brand-500 px-4 py-2 text-sm font-medium text-white"
      >
        Go to the form
      </a>
    </div>
  )
}
```

**"Refrain has not submitted anything and never will — that part is yours."**

Put that sentence in the `attacca` state, permanently. It is the last thing a user reads before
they do the one action that matters. §11 says this is your differentiation; say it out loud in
your own UI, every single time.

---

## Step 7 — Test the human gate

This is the most important test in the project, and it is a **negative** test.

### 7a — Prove Refrain cannot submit

```bash
# Run this before every release. It must return nothing.
grep -rniE "\.click\(\)|submit\(\)|requestSubmit|form\.submit|Enter|dispatchEvent.*keydown" \
  apps/extension packages/vault packages/mapping

# Every hit must be one of:
#   - a widget that only responds to .click() (Chapter 7, documented)
#   - the word "Enter" in copy or an onKeyDown handler
# Anything that clicks a button whose text contains "submit" is a bug.
```

```ts
// apps/extension/entrypoints/sidepanel/machine.test.ts
import { createActor } from "xstate"
import { reviewMachine } from "./machine"

it("has no path from `filled` to a submitted state", () => {
  const actor = createActor(reviewMachine).start()
  actor.send({ type: "SCAN" })
  actor.send({ type: "SCANNED", schema: mockSchema, results: mockResults })
  actor.send({ type: "APPROVE" })
  actor.send({ type: "WROTE" })

  expect(actor.getSnapshot().value).toBe("filled")
  expect(actor.getSnapshot().status).toBe("done")   // final. Cannot leave.
})
```

### 7b — The CAPTCHA test

Put a reCAPTCHA on a test form. Fill everything else. Press approve.

- [ ] Refrain enters `awaiting-human` and **writes nothing**
- [ ] The message says solve it yourself
- [ ] After solving manually and pressing retry, it writes normally
- [ ] No code path attempted to interact with the CAPTCHA iframe

### 7c — The correction rate

**Phase 1's headline metric (§20): correction rate should *decrease* over time.**

```ts
// Count edits the user makes after approving. Logged to the Setlist in Ch.10.
const corrections = results.filter(
  (r) => r.value?.editedByUser && !preApprovalEdits.has(r.schema.id),
).length
```

Log it. Do not build a dashboard. Just a number in the Setlist you can watch fall.

---

## Step 8 — Commit

```bash
git add -A
git commit -m "feat(fermata): review screen, provenance on every row, double gate, xstate machine"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The XState machine, all guards including `pageBlocked` and `captchaPresent`** | **You. Entirely.** It is the product's spine |
| **`ReviewRow` — every decision in it** | **You.** This is the component that matters |
| The `SummaryBar` copy | **You** — words are the design |
| The `Refusal` copy | **You** |
| The `FilledSummary` copy, including the "that part is yours" line | **You** |
| Hover → page highlight wiring | **You** — it is the connection moment, do not delegate the feel |
| Panel CSS (`.row`, `.row--needs-input`, chip shapes) | **OpenCode** from your token file |
| `NeedsAttention`, `ProvenanceLegend`, `FermataFooter` markup | **OpenCode** |
| The `machine.test.ts` human-gate tests | **OpenCode**, then read each and explain why it must fail without the gate |
| A Playwright test that fills a real Google Form and asserts arrival | **OpenCode** |

---

## Gotchas in this chapter

**XState v5 `setup()` API.** v4 used `Machine({...})` with `services` and `guards` inline.
v5 moved to `setup({ types, guards, actions }).createMachine({...})`. Every tutorial is v4. If
`assign` complains about missing types, you are on v5 and your example is v4.

**The panel is blank on open.** A static import that touches `window.top` or `document` at
module scope. Make it a dynamic `import()` inside the action. Check the panel console.

**Chips are invisible.** You forgot `import "@refrain/ui/tokens.css"` in the panel entry.

**Everything is `needs-input`.** `vault.readProfile()` returns an empty profile because the
vault is locked. Check the unlock state — the panel should show the unlock screen, not a
review screen full of blanks.

**Editing a value reverts when you scan again.** `SCANNED` replaces `results` wholesale,
discarding `edits`. Merge them in the assign, or move `edits` above `results` in the context
and read from it.

**"Go to the form" does nothing.** `chrome.tabs.update({ active: true })` with no `windowId`
does not focus a specific tab. Use the `tabId` you captured at scan time.

**The highlight outlines every field at once and looks like a bug.** You added the class at
scan time instead of on hover. Class on hover, class removal on leave.

**Only 12 of 14 rows render.** Two fields share an `id` because the page reused a `name`
attribute and your `cssPath` fallback collided. `id` must be unique per frame — assert it.

**`needs-input` rows steal focus on load.** You set `autoFocus` on every empty input. Only the
first one.

---

## Verify before moving on

- [ ] 14/14 rows render on a real Google Form with real labels
- [ ] Every row has a visible provenance chip **and** an `sr-only` provenance description
- [ ] A row is editable in one tap
- [ ] Low-confidence rows (0.75–0.85) show the lock and fill
- [ ] High-confidence rows do not warn
- [ ] Values in a closed option list become `needs-input`, never a near-match
- [ ] `SummaryBar` reads as one honest sentence with real counts
- [ ] Hovering a row outlines the matching field on the page
- [ ] A `.nic.in` URL shows the refusal screen with **no** override
- [ ] A CAPTCHA page writes **nothing** and says why
- [ ] `filled` is `type: "final"` and the human-gate test passes
- [ ] The grep for submit automation returns nothing
- [ ] Attacca says "that part is yours"
- [ ] Keyboard-only: you can reach, read, and edit every row
- [ ] **One Google Form filled end-to-end and actually submitted. MVP reached.**

---

## Check yourself before Chapter 10

1. **Why does the review happen before the write, and not after?**
2. **Why is `filled` a `final` state instead of a state with a `SUBMIT` transition?**
3. **What breaks if `REVIEW_THRESHOLD` and the `showLock` value are the same number?**
4. **Why does the provenance live in an `sr-only` span as well as a visible chip?**
5. **Why is the refusal screen not allowed to have an override?**
6. **What is the hover-to-highlight interaction doing that the review rows are not?**
7. **How does `editedByUser` become the Phase 1 success metric?**

---

**Next: [Chapter 10 — The Setlist & Verses](./10-the-setlist.md)** — the tracker that gives users a
reason to come back, Encore for re-filling a stale form, Verses for compressing an essay to a
character limit, and export/delete.