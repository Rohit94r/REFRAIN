# Chapter 10 — The Mascot

> **Day 11 · Goal: one geometry, four states, every size, zero external assets.**
>
> The mascot is not decoration. Per §9 it is a **state indicator with four states**, and
> mascot-driven UI is your distribution strategy — the screenshot a student sends to their
> group chat is your marketing. This is a product chapter with an illustration spec in it.

---

## Understand this first

### The mascot has a job, and the job is to disappear

From §9:

> The mascot should make someone smile **once**, then become invisible. If it narrates,
> celebrates, and greets, it fails.

That is a hard constraint and it is easy to violate by accident. Every extra animation, every
"Great job! 🎉", every celebratory confetti on submit is a tax on the one person trying to
finish a scholarship form in thirty seconds.

**Four states. Not five. Not a personality.** Idle, working, needs you, done. That is the whole
vocabulary.

### Design it as one geometry, not four drawings

The single most important decision in this chapter. You have two options:

| | Four separate SVGs | One parameterised geometry |
|---|---|---|
| Files | 4 files to keep in sync | 1 file |
| Proportions | drift apart over time | impossible to drift |
| New states | draw a new illustration | add a row to a table |
| Bundle | 4× the markup | 1× |
| Colour | hard-coded per file | driven by your token |

Four illustrations is a maintenance burden that guarantees inconsistency. **One geometry with a
parameter table is the correct engineering**, and it means adding a fifth state later is a table
row, not an afternoon in a drawing tool.

### The "smiley blob" is a deliberate design choice, not laziness

You need a shape that reads as *warm and friendly* at **20px in a side panel** and at **200px on
a landing page**. That is a hard constraint and it eliminates most character design:

- **Faces with mouths, ears, arms** → unreadable at 20px, and a botched 20px render looks broken
- **Angular / geometric** → reads as corporate. You are §9's "Duolingo, not Salesforce"
- **Detailed / textured** → a 3px blur at 20px

A **soft rounded blob with two eyes** is the only geometry that survives both extremes. It is
the reason Duolingo's owl, Slack's ghost, and Notion's blob all work at small sizes. You are not
being lazy — you are choosing the shape class that solves the constraint.

### Do not make it musical

Your word system is musical (§1): Refrain, Verse, Motif, Fermata, Attacca. It is tempting to
draw a note, a treble clef, or eighth-notes.

**Do not.** A note-shaped mascot reads as a music app. Someone opens your extension expecting
Spotify. The words are internal vocabulary; the character is a companion, not a pun.

**One exception, and it is load-bearing:** the `fermata` state uses the actual fermata glyph —
a dot with an arc above it. That is a *pause symbol*. Every musician and every non-musician
reads it as "held." It is a universal pause mark wearing a music costume, and it is the only
musical reference allowed.

---

## Step 1 — The character bible

Write this down before you draw anything. Without it, you will redraw the eyes next week.

```markdown
# Refrain — character spec v1

## What it is
A soft rounded blob. Two eyes. No mouth in the resting state.
It never speaks. It never celebrates.

## Shape language
- Aspect: slightly wider than tall (1.08:1). Reads as settled, not bouncy.
- Corner radius: maximum. Nothing in the character has a sharp corner.
- Eyes: large relative to the body. Big eyes read as friendly; small read as
  suspicious. Eye diameter ≈ 0.18 of body width.
- Eye spacing: 0.34 of body width. Wider reads as startled; narrower as cross.

## Personality (three words, not a paragraph)
Warm. Patient. Quiet.

## What it never does
- ❌ Smile broadly (teeth, open mouth, grin)
- ❌ Show a mouth in idle
- ❌ Move unless the user is waiting on it
- ❌ Show more than two eyes
- ❌ Use red unless something needs the user
- ❌ Change size

## The one rule
Smile once, then become invisible.
```

> **"No mouth in the resting state" is the important line.** A mouth invites the mascot to talk,
> and a talking mascot narrates, and a narrating mascot fails §9. Eyes carry everything:
> closed arcs read as content, wide circles read as alert, and the fermata arc carries the
> pause. You can build four genuinely distinct states out of eyes plus one ring and no mouth
> at all.

---

## Step 2 — The parameter table

This is the design. Everything else is implementation.

| State | Eyes | Extra | Motion | Brand colour | Word |
|---|---|---|---|---|---|
| `idle` | **closed** arcs, curving up | — | **none** | `brand-300` | Ready when you are |
| `listening` | **open**, wide circles | — | slow pulse, 1.8s loop | `brand-400` | Heard 14 fields on this page |
| `fermata` | **open**, narrower, focused | **fermata glyph** above | ring breath, 2.4s loop | `brand-500` | 2 blanks, 1 attachment |
| `attacca` | **open**, slight forward lean | — | one nudge, 320ms, once | `brand-500` | Filled — review and submit |

Read the motion column: **only `listening` loops forever.** `idle` has none. `fermata` loops
gently but it is a *hold*, so the loop is a breath, not a pulse. `attacca` happens once and
stops. Every animation that loops forever is a tax on someone's attention.

### Colour usage — the mascot has exactly one

```css
/* The character never wears provenance colours. */
.mascot { --mascot-tone: var(--color-brand-400); }
.mascot--fermata { --mascot-tone: var(--color-prov-input); }  /* amber: you are needed */
.mascot--attacca { --mascot-tone: var(--color-brand-500); }
```

**Provenance colours are reserved for provenance chips.** If the mascot turns green to mean
"from your profile" and blue for "from a document," then green and blue stop meaning anything
in the review list. One accent for the character, a strictly separate palette for data meaning.
Two colour systems, zero overlap.

---

## Step 3 — The geometry

One file. Parameterised.

```tsx
// packages/ui/src/mascot/Mascot.tsx
import type { FC } from "react"
import "@refrain/ui/tokens.css"
import "./mascot.css"

export const MascotState = {
  IDLE: "idle",
  LISTENING: "listening",
  FERMATA: "fermata",
  ATTACCA: "attacca",
} as const
export type MascotState = (typeof MascotState)[keyof typeof MascotState]

/** Fixed per §9. The mascot never narrates, celebrates, or greets. */
export const MASCOT_COPY: Record<MascotState, string> = {
  idle: "Ready when you are",
  listening: "Heard 14 fields on this page",
  fermata: "2 blanks, 1 attachment",
  attacca: "Filled — review and submit",
}

/* ── The geometry ────────────────────────────────────────────────────
   One shape. viewBox 48×48 so every size scales from one number.
   Proportions from the character bible: eye diameter 0.18 of body,
   eye spacing 0.34, body 1.08:1.
   ─────────────────────────────────────────────────────────────────── */

interface FaceProps {
  state: MascotState
  /** Render at 1×, 1.5×, or 2× for small surfaces. */
  scale?: 1 | 1.5 | 2
}

const EYE_L = { cx: 16.5, cy: 24 }
const EYE_R = { cx: 31.5, cy: 24 }

function Eyes({ state }: { state: MascotState }) {
  // Closed eyes: an arc curving upward. Reads as content and settled.
  // This is the single most important shape decision in the character.
  if (state === "idle") {
    return (
      <g stroke="var(--mascot-ink)" strokeWidth="2.4" strokeLinecap="round" fill="none">
        <path d="M13.5 25.5q3 -3.5 6 0" />
        <path d="M28.5 25.5q3 -3.5 6 0" />
      </g>
    )
  }

  // Open eyes. fermata narrows the spacing — focus, not alarm.
  const cx = state === "fermata" ? { l: 17.5, r: 30.5 } : { l: EYE_L.cx, r: EYE_R.cx }
  const r = state === "fermata" ? 2.1 : 2.6

  return (
    <g fill="var(--mascot-ink)">
      <circle cx={cx.l} cy={EYE_L.cy} r={r} />
      <circle cx={cx.r} cy={EYE_R.cy} r={r} />
    </g>
  )
}

/**
 * The fermata glyph — a dot under an arc. A universal pause mark
 * wearing a music costume. The ONLY musical reference permitted.
 */
function FermataGlyph() {
  return (
    <g stroke="var(--mascot-ink)" strokeWidth="2" strokeLinecap="round" fill="none">
      <path d="M15 15q9 -8 18 0" />
      <circle cx="24" cy="19.5" r="1.8" fill="var(--mascot-ink)" stroke="none" />
    </g>
  )
}

const Face: FC<FaceProps> = ({ state, scale = 1 }) => (
  <svg
    className="mascot__face"
    viewBox="0 0 48 48"
    width={48 * scale}
    height={48 * scale}
    aria-hidden="true"
    focusable="false"
  >
    {/* Soft highlight. Sells "warm" in two paths, costs nothing. */}
    <ellipse cx="24" cy="25" rx="19" ry="17" fill="var(--mascot-tone)" />
    <ellipse cx="18" cy="17" rx="6" ry="4" fill="white" opacity="0.18" />

    {state === "fermata" && <FermataGlyph />}
    <Eyes state={state} />
  </svg>
)

export interface MascotProps {
  state: MascotState
  /** Counts render as small numerals, never as sentences. */
  ready?: number
  needs?: number
  className?: string
  scale?: 1 | 1.5 | 2
}

export function Mascot({ state, ready, needs, className, scale = 1 }: MascotProps) {
  return (
    <div
      className={`mascot mascot--${state} ${className ?? ""}`}
      // A screen reader announces the state change without interrupting.
      // Not decoration — §11 holds sensitive data, and an inaccessible
      // review screen is a trust failure too.
      role="status"
      aria-live="polite"
    >
      <Face state={state} scale={scale} />
      <p className="mascot__copy">{MASCOT_COPY[state]}</p>

      {state === "fermata" && (
        <span className="mascot__counts">
          <span className="mascot__count mascot__count--ready">{ready ?? 0} ready</span>
          <span className="mascot__count mascot__count--needs">{needs ?? 0} need you</span>
        </span>
      )}
    </div>
  )
}
```

**Six decisions in that file:**

**`viewBox="0 0 48 48"` and nothing else.** Every size — 20px panel, 200px landing page,
favicon — is the same geometry at a different `scale`. You get infinitely many sizes from one
number, and they can never be inconsistent.

**`aria-hidden="true"` on the SVG, `role="status"` on the wrapper.** The shape is decorative;
the *state* is information. A screen reader should announce "Fermata — 2 blanks, 1 attachment"
and skip the drawing.

**`--mascot-ink` is separate from `--mascot-tone`.** The body is brand-warm; the eyes are ink.
Two variables so you can invert for dark mode with one override, instead of editing four paths.

**The highlight is two ellipses, not a gradient.** Gradients at 20px render as mud. A flat
`opacity: 0.18` ellipse survives every size. This is the whole reason the mascot looks
crisp rather than blurry in a side panel.

**`fermata` narrows eye spacing to 17.5/30.5.** A 1px change in coordinates that reads as
*focused* rather than *alarmed*. Alarmed is for errors; the fermata state is not an error, it
is a pause.

**The eye `r` differs (2.6 vs 2.1).** Fermata's smaller eyes plus tighter spacing plus the glyph
above is a **triple signal** — shape, size, and spacing. A colour-blind user in a bright
library gets all three.

---

## Step 4 — Motion

```css
/* packages/ui/src/mascot/mascot.css */

.mascot {
  --mascot-tone: var(--color-brand-300);
  --mascot-ink:   var(--color-brand-900);

  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.75rem;
  padding: 1.5rem;
  border-radius: var(--radius-panel);
  background: var(--color-surface-muted);
  transition: background 200ms ease;
}
.mascot__copy {
  font-size: 0.9375rem;
  color: var(--color-ink);
  text-align: center;
  font-weight: 500;
}

.mascot--listening { --mascot-tone: var(--color-brand-400); }
.mascot--fermata   { --mascot-tone: var(--color-prov-input); }
.mascot--attacca   { --mascot-tone: var(--color-brand-500); }

/* IDLE — not "subtle motion". None. */
.mascot--idle .mascot__face { animation: none; }

/* LISTENING — the only state with a fast loop */
.mascot--listening .mascot__face {
  animation: mascot-pulse 1.8s ease-in-out infinite;
}
@keyframes mascot-pulse {
  0%, 100% { transform: scale(1);    opacity: 0.85; }
  50%      { transform: scale(1.04); opacity: 1; }
}

/* FERMATA — a breath, not a pulse. Slow on purpose: this is a hold. */
.mascot--fermata { position: relative; }
.mascot--fermata::after {
  content: "";
  position: absolute;
  inset: -6px;
  border: 2px solid var(--color-prov-input);
  border-radius: var(--radius-panel);
  animation: mascot-hold 2.4s ease-in-out infinite;
}
@keyframes mascot-hold {
  0%, 100% { opacity: 0.35; transform: scale(1); }
  50%      { opacity: 0.9;  transform: scale(1.015); }
}

/* ATTACCA — one nudge, then still. No confetti, ever. */
.mascot--attacca .mascot__face {
  animation: mascot-nudge 320ms ease-out 1;
}
@keyframes mascot-nudge {
  from { transform: translateY(4px); opacity: 0.6; }
  to   { transform: translateY(0);   opacity: 1; }
}

.mascot__counts { display: flex; gap: 0.5rem; }
.mascot__count {
  font-size: 0.75rem;
  padding: 0.125rem 0.625rem;
  border-radius: var(--radius-chip);
}
.mascot__count--ready { background: var(--color-prov-profile); color: white; }
.mascot__count--needs { background: var(--color-prov-input);   color: var(--color-brand-900); }

/* Dark mode: invert ink, keep tone. Two lines, not a redraw. */
.dark .mascot { --mascot-ink: oklch(0.16 0.02 60); }

/* Non-negotiable. Vestibular disorders are real and the mascot
   animates on every page load. */
@media (prefers-reduced-motion: reduce) {
  .mascot__face,
  .mascot--fermata::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
  }
}
```

### The motion spec, as a table

| State | Duration | Easing | Loop | Why |
|---|---|---|---|---|
| `idle` | 0 | — | — | It is resting. Motion here says "something is happening" and nothing is |
| `listening` | 1.8s | `ease-in-out` | ∞ | The only state where waiting is the point. 1.8s is slow enough not to read as panic |
| `fermata` | 2.4s | `ease-in-out` | ∞ | Deliberately slower than `listening`. A hold should feel slower than a pulse — the timing difference *is* the message |
| `attacca` | 320ms | `ease-out` | **1** | `ease-out` because it decelerates into rest, which reads as "settled." `ease-in` would read as gathering for more |

> **Every looping animation needs a stop condition.** `attacca` plays **once**, because a
> mascot that keeps nudging is asking for attention after the attention is no longer needed.
> That is the "smile once, then become invisible" rule expressed as a duration.

---

## Step 5 — Sizes and surfaces

| Surface | Size | `scale` | Notes |
|---|---|---|---|
| Side panel header | 40px | `1` (48 base, CSS-constrained) | Primary surface. Must read at this size |
| Review screen | 56px | `1.5` | Slightly larger, it is the emotional centre |
| Web app dashboard | 64px | `1.5` | |
| Landing page hero | 200px | rendered via CSS `width` | Same SVG, styled |
| Chrome Web Store screenshots | 128px+ | — | Your distribution asset |
| Favicon / toolbar icon | 16px | — | **Eyes only.** At 16px the body is a blob; two dark dots read. Ship a separate 2-path icon |

**The 16px case is the real test of whether your geometry works.** If it does not read at 16px,
your shape class is wrong and you should change the shape, not add detail.

```tsx
// packages/ui/src/mascot/ToolbarIcon.tsx — the 16px variant
export function ToolbarIcon() {
  return (
    <svg viewBox="0 0 16 16" width="16" height="16" aria-hidden="true">
      <ellipse cx="8" cy="9" rx="6.5" ry="5.5" fill="var(--color-brand-500)" />
      <circle cx="5.8" cy="9" r="1.3" fill="oklch(0.20 0.02 60)" />
      <circle cx="10.2" cy="9" r="1.3" fill="oklch(0.20 0.02 60)" />
    </svg>
  )
}
```

---

## Step 6 — File structure

```
packages/ui/src/mascot/
├── Mascot.tsx           ← the component + the copy table (above)
├── ToolbarIcon.tsx      ← the 16px eyes-only variant
├── mascot.css           ← the motion spec (above)
├── mascot-machine.ts    ← the state machine (Chapter 15 uses it)
└── __tests__/
    └── Mascot.test.tsx
```

**No `.svg` files, no `.png`, no font or image loading.** Everything is inline JSX and CSS.

That is not a purity preference. Three concrete reasons:

1. **No network request.** A mascot loaded from `/assets/mascot.svg` is a request that can fail,
   and it leaks a request to your server in a product whose entire pitch is that nothing leaves
   the device.
2. **Theming works.** `fill="var(--mascot-tone)"` responds to dark mode and to the four state
   tones. An SVG file needs a CSS filter or four copies.
3. **Bundle cost.** Inline JSX for four states of three shapes is ~1.5KB. Four SVG files with
   duplication is 6KB and drifts.

```bash
# Verify you have not regressed
grep -rE "\.svg|\.png|new Image|url\(" packages/ui/src/mascot/ && echo "✗ external asset found" || echo "✓ no external assets"
```

---

## Step 7 — State machine

The mascot's states must be driven by the product, not guessed. This is the reducer; Chapter 15
promotes it into the full XState machine.

```ts
// packages/ui/src/mascot/mascot-machine.ts

export type MascotEvent =
  | { type: "PANEL_OPENED" }
  | { type: "SCAN_STARTED" }
  | { type: "FIELDS_FOUND"; count: number }
  | { type: "MAPPING_DONE"; ready: number; needs: number }
  | { type: "USER_EDITED" }
  | { type: "FILLED"; count: number }
  | { type: "CLEARED" }
  | { type: "LOCKED" }

export interface MascotState2 {
  state: MascotState
  ready: number
  needs: number
}

const INITIAL: MascotState2 = { state: "idle", ready: 0, needs: 0 }

export function mascotReducer(s: MascotState2, e: MascotEvent): MascotState2 {
  switch (e.type) {
    case "PANEL_OPENED":
    case "SCAN_STARTED":
      return { ...s, state: "listening" }

    case "FIELDS_FOUND":
      // Count changes the copy without changing the state.
      return { ...s, state: "listening" }

    case "MAPPING_DONE":
      return { ...s, state: "fermata", ready: e.ready, needs: e.needs }

    case "USER_EDITED":
      // Stays in fermata. Editing does not advance the state — the
      // hold is the hold.
      return s

    case "FILLED":
      return { ...s, state: "attacca", ready: e.count, needs: 0 }

    case "CLEARED":
    case "LOCKED":
      return INITIAL
  }
}
```

**`USER_EDITED` returns `s` unchanged, deliberately.** The obvious "improvement" is to advance to
`attacca` after the user finishes editing. That is wrong — `attacca` means *the values are in the
form*. Editing a value in the review panel has not written anything to the page yet. The mascot
must not imply a state the product has not reached.

---

## Step 8 — Do the actual test

§9: *"Show it to someone. If they say 'oh cute, that's a little blob' and then look back at the
page — it works. If they read the mascot instead of the content — it fails."*

### The playground

```tsx
// apps/web/src/App.tsx — throwaway, delete before Chapter 11
import { useState } from "react"
import { Mascot, type MascotState } from "@refrain/ui"
import "@refrain/ui/tokens.css"

const STATES: MascotState[] = ["idle", "listening", "fermata", "attacca"]

export default function App() {
  const [state, setState] = useState<MascotState>("idle")
  return (
    <main className="min-h-dvh grid place-items-center gap-8 p-12 bg-surface">
      <div className="flex gap-2">
        {STATES.map((s) => (
          <button key={s} onClick={() => setState(s)}
            className={s === state
              ? "px-3 py-1 rounded-full bg-brand-500 text-white"
              : "px-3 py-1 rounded-full border"}>
            {s}
          </button>
        ))}
      </div>

      <Mascot state={state} ready={18} needs={3} />

      {/* Scale ladder — check every size at once */}
      <div className="flex items-end gap-6">
        {[16, 32, 48, 96, 200].map((px) => (
          <div key={px} className="grid place-items-center gap-1">
            <div style={{ width: px, height: px }}>
              <Mascot state={state} />
            </div>
            <span className="text-xs text-ink-muted">{px}px</span>
          </div>
        ))}
      </div>
    </main>
  )
}
```

### The checklist that catches most mascot failures

- [ ] Reads at **16px** without adding detail
- [ ] Reads in **dark mode** with only two CSS overrides
- [ ] All four states are distinguishable **in a greyscale screenshot**
- [ ] `idle` has genuinely zero motion (check with a 5-second screen recording)
- [ ] Reduced-motion emulation in DevTools stops everything
- [ ] A screen reader announces the state change and the copy
- [ ] No `.svg` / `.png` / `url()` in `mascot/`
- [ ] Someone looked at it, said something brief, and **went back to the content**

**The greyscale test is the one people skip and it is the most valuable.** Desaturate a
screenshot of the four states. If `listening` and `fermata` become indistinguishable, the
difference is carried by hue alone — and you just failed 1 in 12 men. Add a shape difference.

---

## Step 9 — Provenance chips

The mascot is the emotional layer. The chips are the **trust layer**, and §9 calls them "the most
designed element in the product." Build them next, same chapter, same discipline.

```tsx
// packages/ui/src/provenance/ProvenanceChip.tsx
import type { Provenance, SourceKind } from "@refrain/fields"
import "@refrain/ui/tokens.css"

const MAP: Record<SourceKind, { glyph: string; tone: string; label: string }> = {
  profile:   { glyph: "✓", tone: "var(--color-prov-profile)",  label: "from your profile" },
  document:  { glyph: "📄", tone: "var(--color-prov-document)", label: "from a document" },
  verse:     { glyph: "✎", tone: "var(--color-prov-verse)",    label: "your verse, adapted" },
  manual:    { glyph: "✎", tone: "var(--color-ink-muted)",     label: "you typed this" },
  inference: { glyph: "◈", tone: "var(--color-ink-muted)",     label: "inferred" },
}

export interface ChipProps {
  provenance?: Provenance
  /** Locked overrides everything. The §11 low-confidence case. */
  locked?: boolean
  editedByUser?: boolean
}

export function ProvenanceChip({ provenance, locked, editedByUser }: ChipProps) {
  if (editedByUser) {
    return <span className="chip chip--manual"><span aria-hidden>✎</span> you</span>
  }
  if (locked) {
    return (
      <span className="chip chip--locked">
        <span aria-hidden>🔒</span> check this
      </span>
    )
  }
  if (!provenance) return null

  const m = MAP[provenance.kind]
  return (
    <span className="chip" style={{ "--chip": m.tone } as React.CSSProperties}
      title={`${m.label} · ${provenance.label}${provenance.locator ? `, ${provenance.locator}` : ""}`}>
      <span aria-hidden>{m.glyph}</span>
      {m.label}
    </span>
  )
}
```

```css
/* packages/ui/src/provenance/chips.css */
.chip {
  display: inline-flex;
  align-items: center;
  gap: 0.375rem;
  font-size: 0.6875rem;
  font-weight: 500;
  line-height: 1;
  padding: 0.25rem 0.5rem;
  border-radius: var(--radius-chip);
  background:  color-mix(in oklch, var(--chip) 14%, transparent);
  color: var(--chip);
  border: 1px solid color-mix(in oklch, var(--chip) 32%, transparent);
  white-space: nowrap;
}

/* SHAPE carries meaning, not just colour. ~1 in 12 men cannot
   rely on the hue channel. */
.chip--manual,
.chip--locked {
  border-radius: 0.375rem;   /* square-ish, not a pill */
  border-width: 1.5px;
}
.chip--locked  { --chip: var(--color-prov-lowconf); }
.chip--manual  { --chip: var(--color-ink-muted); }
```

> **`color-mix()` is the trick that makes the whole system work.** One CSS variable carries the
> hue; the background is that hue at 14%, the border at 32%. Five tokens give you a full
> tappable palette, and **dark mode costs zero extra lines** because the mix inherits the hue
> from whatever the active theme set. Try building this with eight hard-coded hex variants and
> count the maintenance.

`ProvenanceLegend` renders **once**, below the list. Not a tooltip per row — a tooltip is
invisible to a keyboard user and to a phone.

---

## Step 10 — Commit

```bash
git add -A
git commit -m "feat(ui): mascot as one parameterised geometry, 4 states, motion spec, provenance chips"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The character bible** | **You.** Personality is not a code decision |
| **The parameter table** (state × eyes × extra × motion × colour) | **You.** This is the design |
| The SVG geometry and eye coordinates | **You**, then tune by hand — you will want to adjust the spacing |
| `mascot.css` and the motion timing | **You.** Timing is a feel thing |
| `mascotReducer` and the `USER_EDITED` decision | **You** |
| Reasoning about shape-vs-colour for chips | **You** |
| `ToolbarIcon` | **OpenCode** |
| The playground and the scale ladder | **OpenCode** |
| Figma / Illustrator illustration | **Neither.** Three shapes and two eyes. Move on. |
| A snapshot test per state | **OpenCode** |

---

## Gotchas in this chapter

**Every colour turns black after `pnpm install`.** `@refrain/ui` is missing
`"sideEffects": ["**/*.css"]` in its `package.json`. Your bundler sees a package with only
type imports, tree-shakes the whole thing, and your tokens vanish.

**The mascot looks blurry in the side panel.** You set a CSS `width` without a matching
`height`, so the SVG stretches. Always size both, or use `aspect-ratio`.

**Eyes disappear at 20px.** Eye `r` of 2.6 in a 48-unit viewBox is 1px at 20px. Either drop to
a larger eye radius or bump the panel's mascot size. This is the real constraint — check it
first, not last.

**Both states look the same in dark mode.** You set `--mascot-tone` but not `--mascot-ink`,
and dark ink on dark body disappears. Two overrides, and they are not optional.

**The mascot animates forever and you cannot stop it.** You skipped the
`prefers-reduced-motion` block. Go back to Step 4.

**Chips are unreadable in dark mode.** `color-mix()` at 14% on a dark surface is nearly
invisible. Bump to 22% in `.dark`, or the chip reads as a coloured outline with no fill.

**"What is `scale` for?"** It is for the few places you render the SVG at a non-native size
without CSS (a canvas export, an email signature). Most of the time CSS handles it and you
never pass `scale`.

**You drew a musical note.** See "Do not make it musical" above. A note mascot reads as a
music app.

---

## Verify before moving on

- [ ] Reads at 16px, 32px, 48px, 96px, 200px
- [ ] All four states distinguishable in a **greyscale** screenshot
- [ ] `idle` has genuinely zero motion
- [ ] Only `listening` loops fast; `fermata` is slower; `attacca` plays once
- [ ] Reduced-motion emulation stops everything
- [ ] Dark mode correct with two overrides
- [ ] Screen reader announces state + copy, skips the SVG
- [ ] `grep -rE "\.svg|\.png|url\(" packages/ui/src/mascot/` returns nothing
- [ ] Chips work in light and dark, shape-distinct for "you must act"
- [ ] Someone saw it, said one short thing, and went back to the content

---

## Check yourself before Chapter 11

1. **Why is one parameterised geometry better than four SVGs? Give the maintenance reason.**
2. **Why no mouth in the resting state?**
3. **Why must the mascot never use provenance colours?**
4. **Why is `fermata` slower than `listening`?**
5. **What breaks if you drop `sideEffects` from `@refrain/ui`?**
6. **Why does `USER_EDITED` return the state unchanged instead of advancing to `attacca`?**
7. **Why does the shape of a chip matter as much as its colour?**
8. **You want a fifth mascot state. What actually changes?**

---

**Next: [Chapter 11 — The Web App Shell](./11-web-app-shell.md)** — routing, layouts, and every
screen stubbed with real navigation, before any product logic exists. Front end first.