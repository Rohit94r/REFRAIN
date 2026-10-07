# Chapter 3 — The Design System

> **Day 3 · Goal: mascot renders all four states, provenance chips designed.**
>
> The mascot is not decoration. Per §9, it is a **state indicator with four states**, and
> mascot-driven UI is your distribution strategy — the screenshot students share is your
> marketing. So this is a product chapter, not a styling chapter.

---

## Understand this first

### The mascot has a job, and the job is to disappear

From §9:

> The mascot should make someone smile **once**, then become invisible. If it narrates,
> celebrates, and greets, it fails.

That is a hard constraint and it is easy to violate accidentally. Every extra animation,
every "Great job! 🎉", every celebratory confetti on submit is a tax on the one person
trying to finish a scholarship form in thirty seconds.

**Four states. Not five. Not a personality.** Idle, working, needs you, done. That is the
whole vocabulary.

And because your mascot states are named from the musical family, they form a story:

```
Rest  →  Listening  →  Fermata  →  Attacca
rest     listen        HOLD        go on
```

**Fermata is the load-bearing one.** A fermata is a held note — a deliberate pause where the
performer waits for the conductor. That is §11's human gate. Naming it means every surface
that reaches the human gate says the same thing: *this will not move until you move it.*

### Why provenance chips are the hardest design problem in the product

§9 calls them the visual signature: "Make this the most designed element in the product."

That is right, and it is subtle. The review screen is where the user's entire trust is won or
lost. If a chip is noisy, they stop reading them and the whole screen becomes "AI said so."
The chips have to be **scannable in peripheral vision while the user reads the values** —
which means one shape per meaning, one colour per meaning, and zero decoration.

| Chip | Meaning | Shape | Colour |
|---|---|---|---|
| `✓` | from your profile | filled circle | brand green |
| `📄` | extracted from a document | filled circle | brand blue |
| `✎` | your saved Verse, adapted | filled circle | brand purple |
| `⚠` | needs your input | **hollow** triangle | amber |
| `🔒` | low confidence — please verify | **hollow** lock | red |

**The shape change matters more than the colour.** Roughly 1 in 12 men has a red-green
colour vision deficiency, and your users are in a college library on a cheap laptop screen.
Shape is the channel that always works. Never encode meaning in colour alone.

### Tailwind v4 vs v3 — read this before you follow any tutorial

You are on **Tailwind v4**, which is CSS-first. Almost every tutorial, course, and AI
suggestion from before early 2025 shows v3 with a `tailwind.config.js` file. If you follow
one, you will spend an hour confused.

| | v3 | v4 |
|---|---|---|
| Config | `tailwind.config.js` | **CSS, in `@theme`** |
| Content paths | `content: [...]` array | Automatic |
| Import | `@tailwind base/components/utilities` | **`@import "tailwindcss"`** |
| Dark mode | `darkMode: "class"` config | **`@custom-variant dark (&:where(.dark, .dark *))`** |
| Config format | JS object | Any CSS value, including `oklch()` |

**v4 is the default when you run `pnpm add -D tailwindcss`.** Use v4. If a tutorial tells
you to create `tailwind.config.js`, it is wrong for your version.

### `oklch()` — and why warm is a colour decision, not a hue

§9 asks for "warm, rounded, friendly. Duolingo, not Salesforce."

`oklch()` describes a colour perceptually: **L**ightness, **Chroma** (saturation), **hue**.
The advantage over hex is that a whole palette generated at one lightness *looks* consistent
— which is exactly what you need for a brand that must read as warm across 500 UI surfaces.

- `0.68` lightness reads as "friendly"
- `0.52` reads as "professional"
- `0.42` reads as "corporate"

**Pick the brand lightness once, at 0.68, and never break it.** That single decision is most
of what separates "warm companion" from "enterprise dashboard."

---

## Step 1 — Tailwind v4 and the token file

```bash
cd apps/web
pnpm add -D tailwindcss @tailwindcss/vite
pnpm dlx @tailwindcss/cli init -p
```

`vite.config.ts`:

```ts
import { defineConfig } from "vite"
import react from "@vitejs/plugin-react"
import tailwindcss from "@tailwindcss/vite"
import { fileURLToPath, URL } from "node:url"

export default defineConfig({
  plugins: [react(), tailwindcss()],
  resolve: {
    alias: { "@": fileURLToPath(new URL("./src", import.meta.url)) },
  },
})
```

`src/styles/app.css` — **this file is your design system.** Everything visual traces back to it.

```css
@import "tailwindcss";
@import "shadcn/tokens.css";

@custom-variant dark (&:where(.dark, .dark *));

@theme {
  /* ── Brand ramp. One lightness, held. ───────────────────────── */
  --color-brand-50:  oklch(0.97 0.02 75);
  --color-brand-100: oklch(0.94 0.04 72);
  --color-brand-200: oklch(0.89 0.07 70);
  --color-brand-300: oklch(0.83 0.10 68);
  --color-brand-400: oklch(0.77 0.13 66);
  --color-brand-500: oklch(0.68 0.15 62);   /* ← the brand colour. Do not vary this. */
  --color-brand-600: oklch(0.60 0.15 60);
  --color-brand-700: oklch(0.51 0.13 58);
  --color-brand-800: oklch(0.42 0.10 56);
  --color-brand-900: oklch(0.33 0.07 54);

  /* ── Semantic roles, NOT raw colours. ───────────────────────── */
  --color-surface:      oklch(1    0    0);
  --color-surface-muted: oklch(0.97 0.005 75);
  --color-ink:          oklch(0.24 0.02 60);
  --color-ink-muted:    oklch(0.52 0.01 60);

  /* ── Provenance. Five meanings, five hues, all at 0.68 L. ────── */
  --color-prov-profile:  oklch(0.68 0.15 145);  /* green  */
  --color-prov-document: oklch(0.68 0.13 245);  /* blue   */
  --color-prov-verse:    oklch(0.68 0.16 300);  /* purple */
  --color-prov-input:    oklch(0.74 0.15 75);   /* amber  */
  --color-prov-lowconf:  oklch(0.62 0.19 25);   /* red    */

  /* ── Generous radius. "Rounded" is a number. ────────────────── */
  --radius-card: 1.25rem;
  --radius-panel: 1.75rem;
  --radius-chip: 9999px;

  /* ── Soft shadows, never harsh. ─────────────────────────────── */
  --shadow-soft: 0 1px 2px oklch(0.24 0.02 60 / 0.04),
                 0 4px 12px oklch(0.24 0.02 60 / 0.06);
  --shadow-lift: 0 2px 4px oklch(0.24 0.02 60 / 0.05),
                 0 12px 28px oklch(0.24 0.02 60 / 0.10);

  --font-sans: "Figtree", ui-sans-serif, system-ui, sans-serif;
}

.dark {
  --color-surface:      oklch(0.20 0.01 60);
  --color-surface-muted: oklch(0.26 0.012 60);
  --color-ink:          oklch(0.94 0.005 75);
  --color-ink-muted:    oklch(0.68 0.01 65);
}

@layer base {
  * { border-color: var(--color-surface-muted); }
  body {
    background: var(--color-surface);
    color: var(--color-ink);
    font-family: var(--font-sans);
    -webkit-font-smoothing: antialiased;
  }
  :focus-visible {
    outline: 2px solid var(--color-brand-500);
    outline-offset: 2px;
  }
}

@layer utilities {
  /* §9: "Motion while working. Silence while the user types." */
  @media (prefers-reduced-motion: reduce) {
    *, *::before, *::after {
      animation-duration: 0.01ms !important;
      transition-duration: 0.01ms !important;
    }
  }
}
```

> **Two things to notice.** First, components reference `--color-surface` and
> `--color-ink`, never `slate-800`. That is what makes dark mode a nine-line override instead
> of an audit. Second, `prefers-reduced-motion` is not optional — vestibular disorders are
> real and your mascot animates on every page load.

`Figtree` is a warm geometric sans and it is the single highest-leverage font choice in the
product. Self-host it with `@fontsource/figtree` — never load from Google Fonts, because your
privacy claims must survive a network tab being open.

---

## Step 2 — shadcn/ui

```bash
pnpm dlx shadcn@latest init
pnpm dlx shadcn@latest add button card input label badge dialog
```

shadcn copies component source into your repo instead of installing a library. That is the
point: **you own it, and you can make `Button` do exactly what Fermata needs.**

Rename the shadcn tokens to point at your brand ramp, then add only what you need:

- `button` — needs a `variant="fermata"` that is wider and taller than default
- `card` — the review row container
- `input`, `label`, `badge`, `dialog`

**Do not add `select`, `combobox`, or `dropdown-menu` yet.** You do not know what the review
screen needs until Chapter 9. Adding them now is how design systems become 4,000 unused lines.

---

## Step 3 — `packages/ui` structure

```
packages/ui/
├── src/
│   ├── index.ts
│   ├── tokens.css
│   ├── mascot/
│   │   ├── Mascot.tsx
│   │   ├── mascot.css
│   │   ├── MascotIdle.tsx
│   │   ├── MascotListening.tsx
│   │   ├── MascotFermata.tsx
│   │   └── MascotAttacca.tsx
│   ├── provenance/
│   │   ├── ProvenanceChip.tsx
│   │   └── ProvenanceLegend.tsx
│   └── primitives/
│       ├── Button.tsx
│       └── Card.tsx
└── package.json
```

`packages/ui/package.json` — note `sideEffects` and the CSS export:

```json
{
  "name": "@refrain/ui",
  "private": true,
  "type": "module",
  "sideEffects": ["**/*.css"],
  "exports": {
    ".": "./src/index.ts",
    "./tokens.css": "./src/tokens.css"
  },
  "dependencies": {
    "@refrain/fields": "workspace:*",
    "react": "^19.0.0"
  },
  "peerDependencies": { "react": "^19.0.0" }
}
```

`sideEffects: ["**/*.css"]` is required. Without it your bundler sees a package with only
type imports, tree-shakes the whole thing, and **your tokens vanish and every colour in the
app turns black.** This bug appears as "the design system broke and I changed nothing."

---

## Step 4 — The mascot

### `Mascot.tsx`

```tsx
import type { FC } from "react"
import { MascotIdle } from "./MascotIdle"
import { MascotListening } from "./MascotListening"
import { MascotFermata } from "./MascotFermata"
import { MascotAttacca } from "./MascotAttacca"
import "@refrain/ui/tokens.css"
import "./mascot.css"

export const MascotState = {
  IDLE: "idle",
  LISTENING: "listening",
  FERMATA: "fermata",
  ATTACCA: "attacca",
} as const

export type MascotState = (typeof MascotState)[keyof typeof MascotState]

/**
 * Copy is fixed per state. The mascot never narrates, never celebrates,
 * never greets. See §9 — "smile once, then become invisible."
 */
export const MASCOT_COPY: Record<MascotState, string> = {
  idle: "Ready when you are",
  listening: "Heard 14 fields on this page",
  fermata: "2 blanks, 1 attachment",
  attacca: "Filled — review and submit",
}

const REGISTRY: Record<MascotState, FC> = {
  idle: MascotIdle,
  listening: MascotListening,
  fermata: MascotFermata,
  attacca: MascotAttacca,
}

export interface MascotProps {
  state: MascotState
  /** Optional counts. Rendered as small numerals, never as sentences. */
  ready?: number
  needs?: number
  className?: string
}

export function Mascot({ state, ready, needs, className }: MascotProps) {
  const Face = REGISTRY[state]

  return (
    <div className={`mascot mascot--${state} ${className ?? ""}`} role="status" aria-live="polite">
      <Face />
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

> **`role="status"` + `aria-live="polite"`** — a screen reader announces the state change
> without interrupting. This is not decoration. §11 says you hold sensitive student data; an
> inaccessible review screen is a trust failure too.

### `mascot.css` — the motion rules

```css
.mascot {
  display: flex; flex-direction: column; align-items: center; gap: 0.75rem;
  padding: 1.5rem; border-radius: var(--radius-panel);
  background: var(--color-surface-muted);
  transition: background 200ms ease;
}
.mascot__copy {
  font-size: 0.9375rem; color: var(--color-ink); text-align: center; font-weight: 500;
}

/* IDLE — no motion at all. Not subtle motion. None. */
.mascot--idle .mascot__face { animation: none; }

/* LISTENING — the only state that loops */
.mascot--listening .mascot__face { animation: pulse 1.8s ease-in-out infinite; }
@keyframes pulse {
  0%, 100% { transform: scale(1);    opacity: 0.85; }
  50%      { transform: scale(1.04); opacity: 1; }
}

/* FERMATA — deliberately still, with a held ring */
.mascot--fermata .mascot__face { animation: none; }
.mascot--fermata::after {
  content: ""; position: absolute; inset: -6px;
  border: 2px solid var(--color-prov-input);
  border-radius: var(--radius-panel);
  animation: hold 2.4s ease-in-out infinite;
}
@keyframes hold {
  0%, 100% { opacity: 0.35; transform: scale(1); }
  50%      { opacity: 0.9;  transform: scale(1.015); }
}

/* ATTACCA — one short forward motion, then still. No confetti. */
.mascot--attacca .mascot__face { animation: nudge 320ms ease-out 1; }
@keyframes nudge {
  from { transform: translateY(4px); opacity: 0.6; }
  to   { transform: translateY(0);   opacity: 1; }
}

.mascot__counts { display: flex; gap: 0.5rem; }
.mascot__count {
  font-size: 0.75rem; padding: 0.125rem 0.625rem; border-radius: var(--radius-chip);
}
.mascot__count--ready { background: var(--color-prov-profile); color: white; }
.mascot__count--needs { background: var(--color-prov-input); color: var(--color-brand-900); }
```

### The state machine

```tsx
// Never write this by hand inside a component. One source of truth.
export type FormEvent =
  | { type: "PAGE_LOAD" }
  | { type: "FIELDS_FOUND"; count: number }
  | { type: "MAPPING_DONE"; ready: number; needs: number }
  | { type: "USER_EDITED" }
  | { type: "FILLED"; count: number }
  | { type: "PAGE_LEFT" }

const INITIAL = { state: "idle" as MascotState, ready: 0, needs: 0 }

export function mascotReducer(s: typeof INITIAL, e: FormEvent): typeof INITIAL {
  switch (e.type) {
    case "PAGE_LOAD":      return { ...s, state: "listening" }
    case "FIELDS_FOUND":   return { ...s, state: "listening" }
    case "MAPPING_DONE":   return { ...s, state: "fermata", ready: e.ready, needs: e.needs }
    case "USER_EDITED":    return { ...s, state: "fermata" }
    case "FILLED":         return { ...s, state: "attacca" }
    case "PAGE_LEFT":      return INITIAL
  }
}
```

> **This is XState territory in Chapter 9.** For now the reducer is fine and you can read it
> at a glance. When the states start having guards — *"do not enter `attacca` while low
> confidence fields remain"* — that is when you migrate, and you will know exactly why.

### Draw the mascot

Simplest thing that satisfies the test: **one SVG path per state.** Not four illustrations —
four variations on one shape, so it reads as one character.

- **Idle** — closed eyes, still
- **Listening** — eyes open, the pulse animation
- **Fermata** — eyes open, a held ring, no motion
- **Attacca** — a small forward lean

Four tiny SVGs. Do not spend a day in Figma. **A student will see this in a 40px side
panel.** Spend the day on the chips instead.

---

## Step 5 — Provenance chips

```tsx
import type { Provenance, SourceKind } from "@refrain/fields"
import "@refrain/ui/tokens.css"

const MAP: Record<SourceKind, { glyph: string; tone: string; label: string }> = {
  profile:   { glyph: "✓", tone: "var(--color-prov-profile)",  label: "from your profile" },
  document:  { glyph: "📄", tone: "var(--color-prov-document)", label: "extracted from a document" },
  verse:     { glyph: "✎", tone: "var(--color-prov-verse)",    label: "your saved verse, adapted" },
  manual:    { glyph: "✎", tone: "var(--color-ink-muted)",     label: "you typed this" },
  inference: { glyph: "◈", tone: "var(--color-ink-muted)",     label: "inferred" },
}

export interface ChipProps {
  provenance: Provenance
  /** Locked overrides everything. This is the §11 low-confidence case. */
  locked?: boolean
  editedByUser?: boolean
}

export function ProvenanceChip({ provenance, locked, editedByUser }: ChipProps) {
  if (editedByUser) {
    return (
      <span className="chip chip--manual" title="You changed this">
        <span aria-hidden>✎</span> you
      </span>
    )
  }
  if (locked) {
    return (
      <span className="chip chip--locked" title="Low confidence — please verify">
        <span aria-hidden>🔒</span> check
      </span>
    )
  }
  const m = MAP[provenance.kind]
  return (
    <span className="chip" style={{ "--chip": m.tone } as React.CSSProperties} title={`${m.label} · ${provenance.label}`}>
      <span aria-hidden>{m.glyph}</span>
      {m.label}
    </span>
  )
}
```

`chips.css` — **shape carries the meaning:**

```css
.chip {
  display: inline-flex; align-items: center; gap: 0.375rem;
  font-size: 0.6875rem; font-weight: 500; line-height: 1;
  padding: 0.25rem 0.5rem; border-radius: var(--radius-chip);
  background: color-mix(in oklch, var(--chip) 14%, transparent);
  color: var(--chip);
  border: 1px solid color-mix(in oklch, var(--chip) 32%, transparent);
  white-space: nowrap;
}
/* Distinct SHAPE for the two that mean "you must act". */
.chip--input, .chip--locked {
  border-radius: 0.375rem;          /* square-ish, not a pill */
  border-width: 1.5px;
}
.chip--locked { --chip: var(--color-prov-lowconf); }
.chip--manual { --chip: var(--color-ink-muted); }
```

> **`color-mix()` is the trick that makes this work.** One CSS variable carries the hue; the
> background is that hue at 14% and the border at 32%. You get a full tappable palette from
> five tokens, and dark mode works with zero extra code because the mix inherits the hue.

`ProvenanceLegend.tsx` — the static legend under the review screen. Render it **once**, below
the list, not in a tooltip per row.

---

## Step 6 — Wire it up and look at it

In `apps/web/src/App.tsx`, build a throwaway state playground:

```tsx
import { useState } from "react"
import { Mascot, MascotState } from "@refrain/ui"
import { ProvenanceChip } from "@refrain/ui/provenance"
import "@refrain/ui/tokens.css"

const STATES: MascotState[] = ["idle", "listening", "fermata", "attacca"]

export default function App() {
  const [state, setState] = useState<MascotState>("idle")
  return (
    <main className="min-h-dvh grid place-items-center gap-8 p-12 bg-surface">
      <div className="flex gap-2">
        {STATES.map((s) => (
          <button key={s} onClick={() => setState(s)}
            className={s === state ? "px-3 py-1 rounded-full bg-brand-500 text-white" : "px-3 py-1 rounded-full border"}>
            {s}
          </button>
        ))}
      </div>
      <Mascot state={state} ready={18} needs={3} />
      <div className="flex gap-2">
        <ProvenanceChip provenance={{ kind: "profile", label: "you", extractedAt: new Date().toISOString() }} />
        <ProvenanceChip provenance={{ kind: "document", label: "marksheet.pdf", locator: "p.2", extractedAt: new Date().toISOString() }} />
        <ProvenanceChip locked />
      </div>
    </main>
  )
}
```

```bash
pnpm --filter @refrain/web dev
```

**Now do the actual test from §9.** Show it to someone. If they say "oh cute, that's a
little owl" and then look back at the page — **it works.** If they read the mascot instead of
the content — it fails, and you simplify.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "feat(ui): tailwind v4 tokens, mascot 4 states (Rest/Listening/Fermata/Attacca), provenance chips"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| `app.css` — every token, the `oklch` ramp, dark mode overrides | **You.** This is your design system |
| The mascot state machine and `mascotReducer` | **You** |
| `ProvenanceChip` component + the shape-vs-colour reasoning | **You** |
| Four mascot SVG paths | **OpenCode** — then edit them by hand to match your taste |
| shadcn init and component installation | **OpenCode** |
| The `App.tsx` playground | **OpenCode** — then delete it before Chapter 4 |
| Figma mascot design | **Neither.** Four tiny SVGs. Move on. |

---

## Gotchas in this chapter

**Every colour turns black after `pnpm install`.** `packages/ui` is missing
`"sideEffects": ["**/*.css"]`. See Step 3.

**Your brand colour looks different on your laptop than the design file.** `oklch()` is
perceptual and colour-managed displays render it differently from uncalibrated ones. Pick
the lightness, check it on the *worst* screen you own, and stop fiddling.

**`bg-brand-500` does not exist.** Tailwind v4 generates utilities from `--color-*` tokens in
`@theme`. A token named `--color-brand-500` inside `@theme` gives you `bg-brand-500`. If you
put the tokens in `:root` instead of `@theme`, the utilities are never generated.

**Dark mode only half works.** You used a raw colour like `text-slate-800` somewhere instead
of `text-ink`. Grep for `-slate-`, `-gray-`, `-zinc-`, `-neutral-` and replace every one.

**A tutorial told you to create `tailwind.config.js`.** You are on v4. Delete it. Config
lives in CSS now.

**The mascot animates and you cannot stop it.** You skipped the `prefers-reduced-motion`
block in `app.css`. Go back.

---

## Verify before moving on

- [ ] `pnpm --filter @refrain/web dev` renders the playground
- [ ] All four mascot states render, and Fermata is visibly distinct in shape
- [ ] Toggle dark mode — every surface is correct, no raw slate colours remain
- [ ] Chips are distinguishable in greyscale (screenshot, desaturate, check)
- [ ] Reduced-motion emulation in DevTools stops all animation
- [ ] Screen reader announces mascot state changes
- [ ] Someone looked at it and went back to the content

---

## Check yourself before Chapter 4

1. **Why does the mascot have exactly four states? What breaks if you add a fifth?**
2. **Why do the two "you must act" chips have a different shape rather than just a different colour?**
3. **What happens if `@refrain/ui` loses `sideEffects`?**
4. **Why is `role="status"` on the mascot an accessibility requirement, not a nicety?**
5. **A component in your review screen uses `bg-slate-100`. What breaks in dark mode, and why?**
6. **Where does `--color-brand-500`'s lightness of 0.68 come from? What if you made it 0.45?**

---

**Next: [Chapter 4 — The Mascot](./04-the-mascot.md)** — one geometry, four states, every size,
zero external assets.
