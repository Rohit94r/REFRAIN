# Chapter 18 — Production Hardening

> **Day 22 · Goal: know your numbers, pass an accessibility audit, and understand what breaks
> at 10,000 users.**
>
> The chapter between "it works" and "it works for someone else on a bad phone."

---

## Understand this first

### This chapter is about surfaces you cannot see failing

The three bugs in Chapter 7 produce an obvious symptom: *"filled but nothing arrived."* You find
them in an afternoon.

The problems in this chapter produce **no symptom at all**:

- The panel takes 900ms to open. Nobody complains — they just think Chrome is slow.
- Your accent colour fails contrast at 4.2:1. Nobody complains — 7% of men just read the page
  harder and do not know why.
- The bundle is 14MB because you forgot `pdfjs-dist` was un-split. Install conversion drops
  and you blame the Web Store.

**Silent degradation is worse than an outage.** An outage gets fixed. A 900ms panel gets
tolerated and then uninstalled, and you never learn why.

So this chapter is mostly about **establishing numbers, then defending them in CI.** A budget
with no gate is a wish.

---

## Part A — Performance

### A1 — The bundle budget

```bash
# scripts/bundle-budget.sh — runs in CI, fails the build.
set -uo pipefail
BUDGET_KB=15000
ACTUAL=$(du -sk apps/extension/.output/chrome-mv3 | cut -f1)

printf "extension bundle: %s KB / %s KB\n" "$ACTUAL" "$BUDGET_KB"
if [ "$ACTUAL" -gt "$BUDGET_KB" ]; then
  echo "✗ over budget by $((ACTUAL - BUDGET_KB)) KB"
  # Show the biggest offenders. Almost always one dependency.
  du -sk apps/extension/.output/chrome-mv3/* | sort -rn | head -10
  exit 1
fi
echo "✓ within budget"
```

```jsonc
// Better: per-package budgets, so you know WHICH one grew.
{
  "budgets": [
    { "path": "apps/extension",           "maxKb": 15000, "note": "includes pdf.js + tesseract" },
    { "path": "apps/web",                 "maxKb":  450 },
    { "path": "packages/fields",          "maxKb":    2 },
    { "path": "packages/vault",           "maxKb":   45 },
    { "path": "packages/ui",              "maxKb":   60 },
    { "path": "packages/mapping",         "maxKb":   35 },
    { "path": "apps/api",                 "maxKb":  250, "note": "server, node_modules excluded" }
  ]
}
```

> **Budget the extension at 15MB and know why it is not 3MB.** Two dependencies dominate:
> `pdfjs-dist` (~1.2MB) and Tesseract's core WASM plus the English model (~8MB). That 8MB is
> non-negotiable, and understanding *why* is the point:
>
> **MV3 forbids remotely hosted executable code.** Tesseract's core is a WASM module — that is
> executable code. You cannot fetch it from a CDN at runtime, no matter how you phrase the URL.
> It must be inside the zip.
>
> This is why 15MB is correct and "optimise it" is not the answer. The right response is the one
> in the next section.

### A2 — Bundle it, load it lazily

Bundled and loaded are different questions. **In the zip: yes. On startup: no.**

```ts
// packages/extract/src/index.ts
// Nothing heavy at module scope. A bare `import` here would pull
// Tesseract into the side panel's initial chunk.
export const extractText = (p: string) => import("./pdf").then((m) => m.readPdf(p))
export const ocr = (c: HTMLCanvasElement[]) => import("./ocr").then((m) => m.ocrDocument(c))
export const extractFacts = (p: unknown) => import("./rules").then((m) => m.extractFacts(p))
```

```ts
// Verified by a test, not by inspection.
it("the side panel's initial chunk does not contain tesseract", async () => {
  const panelEntry = await import("../../apps/extension/entrypoints/sidepanel/index.ts")
  const loaded = Object.keys(panelEntry)
  expect(loaded).not.toContain("ocr")
})
```

**The result:** the zip is 15MB, but the panel opens in 300ms because the 8MB is only read off
disk when someone drops in a photographed marksheet.

> **This is the distinction most people miss.** "Bundle size" that matters is *what the user
> downloads once* and *what the browser parses on startup*. A 15MB extension whose heavy
> dependency is code-split is faster to open than a 2MB extension that parses everything on
> every panel open. Measure the parse time, not the zip size, when you care about speed.

### A3 — Panel cold start

```ts
// apps/extension/entrypoints/sidepanel/perf.ts
export const PANEL_BUDGETS = {
  /** Panel UI interactive. The number a user actually feels. */
  interactive: 400,
  /** Vault ready: passphrase derived, key in hand. */
  vaultReady: 800,
  /** First scan result on a real portal. */
  firstScan: 250,
} as const
```

```ts
// Measure it in the browser, not in a laptop terminal.
performance.mark("panel:start")
document.addEventListener("DOMContentLoaded", () => {
  performance.mark("panel:dom")
  performance.measure("panel-interactive", "panel:start", "panel:dom")
})

// The vault is the slow part and it is deliberate.
const t0 = performance.now()
await vault.unlock(passphrase)
console.info(`vault ready in ${performance.now() - t0}ms`)
```

**Where the milliseconds actually go:**

| Cost | Typical | Reducible? |
|---|---|---|
| JS parse + execute | 40–90ms | Only by shipping less JS |
| React mount | 30–60ms | Barely. This is React's floor |
| Token/CSSOM | 20–50ms | Only by shipping less CSS |
| **PBKDF2 600k** | **180–350ms** | **No — and that is the point** |
| IndexedDB open | 10–30ms | No |

> **The vault unlock dominates, and it is supposed to.** PBKDF2 at 600k iterations is ~250ms on a
> modern laptop and ~900ms on a ₹12,000 Android phone. **Do not reduce it to make a benchmark
> look good.** If it is slow on low-end hardware, the honest response is to show progress and
> keep the cost — a vault that unlocks in 20ms is a vault with 6,000 iterations, and 6,000 is
> crackable.
>
> The correct fix for perceived slowness is a **visible progress state** during derivation, not
> a weaker KDF. Chapter 6's `status: "deriving"` exists for exactly this.

```tsx
// What the user sees while PBKDF2 runs.
{vault.status === "deriving" && (
  <div role="status" aria-live="polite">
    <progress className="w-full" aria-label="Unlocking your vault" />
    <p className="mt-2 text-xs text-ink-muted">Unlocking — this takes a moment on purpose.</p>
  </div>
)}
```

> **"This takes a moment on purpose" is worth saying out loud.** A user who waits 300ms for
> something with no explanation assumes something is wrong. One sentence converts a perceived
> hang into a perceived feature, and it teaches the user why the product is safe — which is
> marketing that happens inside the product.

### A4 — The content script must not slow the page

Your content script runs on **every page**. It has a hard budget of ~30ms and it must be
completely invisible.

```ts
// apps/extension/entrypoints/refrain.content/index.ts

// 1. Bail before doing anything, if this page cannot be a form.
if (!couldBeAForm(location.href, document)) return

// 2. Run idle. Never block parsing or paint.
requestIdleCallback(() => scan(document), { timeout: 500 })

// 3. Yield between frames. A 30-frame iframe form would otherwise
//    freeze the tab for 300ms.
for (const [i, frame] of frames.entries()) {
  if (i % 5 === 0) await new Promise((r) => setTimeout(r, 0))
  await scanFrame(frame)
}
```

```ts
/** Cheap, and it excludes most of the web in one regex. */
function couldBeAForm(url: string, doc: Document): boolean {
  if (doc.querySelector("form")) return true
  if (doc.querySelector("input, select, textarea")) return true
  // Role attribute — the accessible, framework-agnostic signal.
  if (doc.querySelector('[role="textbox"], [role="combobox"], [role="radio"]')) return true
  // Only bother on pages that even mention forms.
  return /\/(apply|signup|register|admission|form|survey|job|internship)/i.test(url)
}
```

**And measure it on the real pages you care about:**

```bash
# The performance number that matters: does the page feel slower?
# Load a real Google Form with the extension enabled, and compare
# FCP against the same page without it. The delta must be < 50ms.
```

> **A content script that adds 200ms to page load is a bug that will cost you reviews.** Users
> do not report "your extension made my browser slow." They uninstall it and forget why. And
> `requestIdleCallback` is not an optimisation here — it is the difference between a tool people
> keep and a tool people disable, which is the only real metric for browser extensions.
>
> **`setTimeout(r, 0)` between frame batches is the fix for multi-frame portals.** Chapter 7
> taught `allFrames: true` and it is non-negotiable for correctness. The yield is what makes it
> survivable — without it, scanning a 30-frame embedded form is a visible stall.

### A5 — IndexedDB and the profile

```ts
// Profile read: every panel open, every page scan. Measure it.
console.time("profile:read")
const profile = await readProfile(key)
console.timeEnd("profile:read")   // target: < 15ms for 40 facts
```

| Operation | Target | If you are over it |
|---|---|---|
| Read 40 facts | < 15ms | Missing index on `key`. Check `explain()` |
| Write one fact | < 5ms | Fine. Do not batch single writes |
| Full profile export | < 200ms | Fine |
| Migration on upgrade | < 500ms | Too slow — split into chunks, `setTimeout(0)` between |
| Search across 500 facts | < 50ms | Client-side index, see below |

```ts
/** Chapter 13 killed server-side search. So search happens here. */
export function buildSearchIndex(facts: ProfileFact[]): Map<string, Set<string>> {
  const index = new Map<string, Set<string>>()
  for (const f of facts) {
    for (const token of `${f.key} ${f.value}`.toLowerCase().split(/\W+/)) {
      if (!token) continue
      ;(index.get(token) ?? index.set(token, new Set()).get(token)!).add(f.key)
    }
  }
  return index
}
```

> **A 40-fact vault needs no index and does not want one** — `filter()` over 40 strings is under
> a millisecond, and an index is a cache you must invalidate on every write. **Build the index
> above ~200 facts**, and rebuild it on vault unlock rather than incrementally on write. A
> 200ms rebuild at unlock is invisible; an invalidation bug that returns stale results is a
> trust failure.

---

## Part B — Accessibility

§11 holds a student's marksheet, CGPA, and address. **An inaccessible review screen is not a
usability problem here — it is a privacy and fairness problem**, because the person most likely
to need it is the one with the most difficulty using it.

### B1 — The automated gate

```bash
pnpm --filter @refrain/web dlx @axe-core/cli https://refrain.dev \
  --exit   # non-zero on violation. Fails CI.
```

```yaml
- name: Accessibility
  run: pnpm dlx @axe-core/cli http://localhost:5173 --exit --tags wcag2a,wcag2aa
```

**Automation catches roughly 30% of issues.** The rest need a human and a keyboard.

### B2 — The keyboard audit

**Do this on every interactive surface. It takes ten minutes and finds things no tool does.**

```tsx
// 1. Every interactive element must be reachable and activatable.
<button onClick={submit}>Fill 14 fields</button>   // ✅ native, focusable, Enter/Space

<div onClick={submit}>Fill 14 fields</div>        // ❌ not focusable, no keyboard
```

> **In Chapter 9 you made the fill button `type: "final"`.** Now verify the *keyboard* path:
> tab to it, press Enter. It must work identically to a click. A gate that cannot be reached
> by keyboard is a gate that excludes keyboard users from correcting their own data — which for
> §11's user is the most important thing on the screen.

### B3 — Focus management in the review list

This is where a 40-row review list is genuinely hard, and where naive implementations break.

```tsx
function ReviewRow({ fact, isFirst, isLast }: Props) {
  const ref = useRef<HTMLInputElement>(null)

  // On unmount (row removed or promoted), move focus somewhere
  // real. Losing focus to <body> sends a screen reader user back
  // to the top of the document.
  useEffect(() => () => {
    const next = document.querySelector<HTMLElement>(
      '[data-review-row]:not([data-done="true"]) input, [data-fill-button]',
    )
    next?.focus()
  }, [])

  return (
    <div data-review-row data-done={fact.status === "ready"}>
      <label htmlFor={`fact-${fact.key}`} className="sr-only">{humanLabel(fact.key)}</label>
      <input id={`fact-${fact.key}`} ref={ref} value={fact.value} /* … */ />

      <button
        onClick={() => promote(fact.key)}
        disabled={fact.status === "ready"}
        aria-label={fact.status === "ready"
          ? `${humanLabel(fact.key)} is filled`
          : `Fill ${humanLabel(fact.key)} with ${fact.value}`}
      >
        {fact.status === "ready" ? <CheckIcon /> : <ArrowIcon />}
      </button>
    </div>
  )
}
```

**The five rules, and every one has a real failure mode:**

| Rule | Failure without it |
|---|---|
| `<label htmlFor>` on every input | Screen reader announces "edit text" 40 times |
| `aria-label` describing the *action*, not the object | "button button button" |
| `sr-only` for redundant visible text | Duplicated announcements |
| Focus moves on row removal | Focus lost to `<body>`, user stranded |
| Escape / arrow keys work in the list | 40 tabs per review pass |

```css
/* A visible focus ring that survives your tokens. Never `outline: none`
   without a replacement — that is the most common a11y regression. */
:focus-visible {
  outline: 2px solid var(--color-brand-600);
  outline-offset: 2px;
  border-radius: var(--radius-chip);
}
@media (prefers-reduced-motion: no-preference) {
  :focus-visible { transition: outline-offset 120ms ease; }
}
```

> **Removing an outline without replacing it makes your app unusable by keyboard and fails WCAG
> 2.4.7 outright.** Chapter 3's design-system reset probably contains `*:focus { outline: none }`
> — it is in half of all Tailwind setups. **Check for it now.** `:focus-visible` gives you the
> ring only for keyboard interaction, which is better than either extreme.

### B4 — Colour contrast, and the chips specifically

Chapter 3's chips are `color-mix()`ed provenance colours at 14% background. **That is where
contrast fails.**

```bash
# Compute it. Do not eyeball it.
pnpm dlx @axe-core/cli https://refrain.dev --rules color-contrast
```

| Pair | Target | Why |
|---|---|---|
| Body text on surface | **4.5:1** | AA, normal text |
| Muted text on surface | **4.5:1** | "Muted" is where accessibility dies |
| Chip text on chip fill | **4.5:1** | `color-mix` at 14% will not get there |
| Focus ring on any background | **3:1** | Non-text contrast, WCAG 1.4.11 |
| Mascot eyes on body | **3:1** | Shape, not just colour |

```css
/* The fix for 14% chips: raise the fill AND use a darker text. */
.chip {
  background: color-mix(in oklch, var(--chip) 18%, var(--color-surface));
  color:       color-mix(in oklch, var(--chip) 82%, var(--color-ink));
  border: 1px solid color-mix(in oklch, var(--chip) 45%, transparent);
}
.dark .chip {
  background: color-mix(in oklch, var(--chip) 22%, var(--color-surface));
  color:       color-mix(in oklch, var(--chip) 88%, white);
}
```

> **Provenance colour must survive both greyscale and low vision.** Chapter 3 already made shape
> carry meaning for the "you must act" chips. This is the other half: the *text inside* the chip
> has to be legible, and 14% on a light surface routinely lands at 2.8:1 — which is decorative
> under WCAG, not readable. Mix against the surface, not against transparency, and mix the text
> toward the ink.

### B5 — Motion, again, with teeth

```css
/* Chapter 4 wrote this. Verify it actually applies to the
   review screen's transitions too, not just the mascot. */
@media (prefers-reduced-motion: reduce) {
  *, *::before, *::after {
    animation-duration: 0.01ms !important;
    animation-iteration-count: 1 !important;
    transition-duration: 0.01ms !important;
    scroll-behavior: auto !important;
  }
}
```

> **The blanket rule, not the targeted one.** Chapter 4 disabled motion on the mascot
> specifically, which is correct but insufficient — the review screen's slide-ins, the chip
> transitions, and any future animation are all still moving. A vestibular disorder is triggered
> by *large-area* motion, and the review list is the largest animated surface in the product.
> Use the blanket rule in your global stylesheet and keep the targeted one for documentation.

### B6 — Screen reader smoke test

**Do this once, properly. It is the single highest-value accessibility activity.**

Turn on VoiceOver (macOS: `Cmd+F5`) or NVDA (Windows: `Ctrl+Alt+N`) and walk one full flow:
scan → review → correct a value → fill → confirm.

```ts
// Assert the live regions exist and are polite.
expect(screen.getByRole("status")).toHaveAttribute("aria-live", "polite")
expect(screen.getByText("2 fields need you")).toBeInTheDocument()
```

**What you are listening for, and what each failure means:**

| You hear | Problem |
|---|---|
| "edit text, edit text, edit text" ×40 | Missing labels |
| Nothing when a value changes | Missing `aria-live` |
| "checkbox, checked" for a fill | `<div role="checkbox">` instead of `<input type="checkbox">` |
| The mascot's alt text | SVG not `aria-hidden` |
| "button" with no name | Icon button without `aria-label` |

> **The last one is the most common.** Chapter 9's per-row fill button is an icon. Without an
> `aria-label` that names the *action and the value*, VoiceOver announces "button" forty times
> and the review screen becomes unusable. The `aria-label` in B3 is not a nicety — it is the
> difference between a screen reader user reviewing their own data and giving up.

---

## Part C — Security

### C1 — The web app's CSP

```ts
// apps/web — Vite dev server headers, and the host config for prod.
const csp = [
  "default-src 'self'",
  // Vite injects styles with inline <style> in dev.
  "style-src 'self' 'unsafe-inline'",
  // React 19 does not need unsafe-eval. If you see it required,
  // something is using eval() and that is a bug, not a config need.
  "script-src 'self'",
  "img-src 'self' data: blob:",
  // blob: for the object URLs you create in the PDF viewer.
  "connect-src 'self' https://api.refrain.dev",
  "font-src 'self'",
  "object-src 'none'",        // <embed>/<object>. Kills plugin XSS vectors.
  "base-uri 'self'",
  "form-action 'self'",
  "frame-ancestors 'none'",   // clickjacking
  "upgrade-insecure-requests",
].join("; ")
```

```nginx
# nginx, if you front the web app with anything
add_header Content-Security-Policy "$CSP" always;
add_header X-Content-Type-Options    "nosniff"            always;
add_header Referrer-Policy           "strict-origin-when-cross-origin" always;
add_header X-Frame-Options           "DENY"               always;
add_header Permissions-Policy        "geolocation=(), camera=(), microphone=(), interest-cohort=()" always;
add_header Strict-Transport-Security "max-age=63072000; includeSubDomains; preload" always;
```

> **`frame-ancestors 'none'` is a security control, not a formality.** Without it, someone
> iframes your review screen on a phishing page, overlays a fake "Confirm" button, and harvests
> whatever the user types. Your review screen *displays their name, CGPA, and address* — a
> high-value clickjacking target that generic advice never considers.
>
> **And `object-src 'none'`** matters here more than in most apps, because Chapter 11 renders
> user-supplied PDFs and images. `blob:` in `img-src` is the legitimate use; `<object>` is not.

### C2 — The extension's CSP is stricter

```jsonc
// apps/extension/entrypoints/...  — WXT config, or manifest notes
{
  "content_security_policy": {
    // MV3 extension pages default to this and you cannot loosen it.
    "extension_pages": "script-src 'self'; object-src 'self'"
  }
}
```

```ts
// WXT config
export default defineConfig({
  manifest: {
    // No remote code. Ever. MV3 rejects the extension otherwise.
    content_security_policy: {
      extension_pages: "script-src 'self'; object-src 'self'",
    },
    // Your own domains only. <all_urls> triggers a review flag.
    host_permissions: [
      "https://forms.google.com/*",
      "https://*.naukri.com/*",
      "https://*.internshala.com/*",
      "http://localhost/*",       // local form development. Remove at publish time.
    ],
  },
})
```

> **`host_permissions` is the single biggest determinant of whether your extension gets
> rejected, and it is worth being surgical.** `<all_urls>` works and it will draw a review flag
> plus a user prompt that reads like a malware warning. Listing real form domains is better for
> review *and* for trust — the permission prompt becomes "reads and changes data on 5 sites"
> instead of "reads and changes all your data on all websites."
>
> **`http://localhost/*` must be removed before you publish.** It is genuinely useful for
> development and it is a review red flag, because localhost access is how credential-stealing
> extensions work. Remove it in the release workflow and verify the published manifest.

### C3 — API security headers and input limits

```ts
// The three that actually matter for a JSON API.
app.use("*", secureHeaders(), {
  contentSecurityPolicy: false,   // irrelevant for JSON
  strictTransportSecurity: "max-age=63072000; includeSubDomains",
  xFrameOptions: "DENY",
})
```

```ts
// Request body size. Before it reaches a route handler.
app.use("*", async (c, next) => {
  const len = Number(c.req.header("content-length") ?? 0)
  if (len > 12 * 1024 * 1024) {              // 12MB > the 10MB blob cap
    throw new AppError("payload_too_large", "That file is too large.", 413)
  }
  await next()
})
```

> **`content-length` is advisory and can be absent.** A chunked request has none, so this check
> is a cheap early rejection, not a defence. The real limit is per-blob `byteLength` in
> Chapter 16's Zod schema plus the Hono/Node body limit. **Have both.** A missing content-length
> header must not be a way to upload 500MB.

```ts
// Validate every body against a schema. Always. Including ones
// "that only we send".
const RegisterRequest = z.object({
  email: z.string().email().max(254),          // max, not just .email()
  authKey: z.string().regex(/^[A-Za-z0-9_-]{43}$/),
}).strict()                                    // strict on the API boundary
```

> **`.strict()` on API input, `.nonstrict()` on extension messages.** Chapter 17's rule inverted,
> and both halves matter: a public API should reject unknown fields loudly (a typo in a client
> becomes a 422 you can see), while a cross-boundary message to an extension you cannot force to
> update must strip unknowns gracefully.

### C4 — Rate limiting as abuse prevention

```ts
// Beyond Chapter 12's limits: per-account and per-device caps.
const SYNC_LIMITS = {
  bytesPerHour: 50 * 1024 * 1024,   // ~50MB/hour. A normal week is < 1MB.
  blobCount: 5000,                  // hard ceiling. A person has maybe 200 documents.
}
```

```ts
async function assertWithinQuota(userId: string) {
  const [{ total }] = await BlobModel.aggregate([
    { $match: { userId: new ObjectId(userId), deletedAt: null } },
    { $group: { _id: null, total: { $sum: "$byteLength" } } },
  ])
  if ((total ?? 0) > SYNC_LIMITS.bytesPerHour * 24 * 30) {
    throw new AppError("payload_too_large", "Storage limit reached. Delete a document.", 413)
  }
}
```

> **Abuse from an authenticated user is still abuse.** A script that creates 100,000 blobs
> costs you money and makes Atlas's free tier unusable. A quota is the difference between
> "someone is using my API" and "someone is using my API and I am paying for it."
>
> **Put the quota number in Settings from day one.** Chapter 5's stub said "A decrypted archive,
> on your machine" next to the delete row. Add a "2.1 MB of 1 GB used" line next to it. Users
> do not discover limits by hitting them.

---

## Part D — Capacity

### Load testing that matches reality

```ts
// apps/api/test/load.test.ts — a realistic shape, not a synthetic one.
describe("sync under load", () => {
  it("serves a 500-document profile pull in under 2s", async () => {
    await seedUser({ facts: 500, documents: 40, bytes: 45 * 1024 * 1024 })

    const t0 = performance.now()
    const pull = await call("GET", "/sync/pull?since=0", {}, { token })
    const metadataMs = performance.now() - t0

    const ids = (await pull.json()).entries.map((e: SyncEntry) => e.docId)
    const t1 = performance.now()
    await call("POST", "/sync/blobs", { docIds: ids.slice(0, 100) }, { token })
    const blobsMs = performance.now() - t1

    // Metadata pull is the common case. It must stay tiny.
    expect(JSON.stringify((await pull.clone().json())).length).toBeLessThan(60_000)
    expect(metadataMs).toBeLessThan(400)
    expect(blobsMs).toBeLessThan(1500)
  })

  it("assigns unique revisions under 50 concurrent pushes", async () => {
    const results = await Promise.all(
      Array.from({ length: 50 }, (_, i) =>
        call("POST", "/sync/push", { syncVersion: 1, changes: [makeChange(`fact-${i}`)] }, { token })),
    )
    expect(results.every((r) => r.status === 200)).toBe(true)
    // The Chapter 15 race, asserted at scale.
    const revisions = results.map((r) => r.revision)
    expect(new Set(revisions).size).toBe(50)
  })
})
```

### The capacity numbers you are actually sized for

| Users | API | MongoDB | Notes |
|---|---|---|---|
| 100 | 1 instance, M0 | M0 free | Free forever. Do not optimise |
| 1,000 | 1 instance, M10 | M10 | The real starting point for backups |
| 10,000 | 2 instances | M30 | Replica set, PITR, alerting |
| 100,000 | Autoscale + CDN | M80 | Post-Phase-2, at a real trigger |

> **Build for 1,000 users and make it trivial to grow.** A solo project with 100 users does not
> need a cache layer, a read replica, or a queue. **Complexity you add before you need it is
> complexity you maintain while debugging something else.** Chapter 12's decision to not require
> Redis for local dev is the same principle.
>
> **The number worth knowing is the free tier's cliff.** Atlas M0 is free and fine until you
> cross roughly 5GB or need backups, and then you wake up to a bill. **Put an alert on storage
> used** so the transition is a notification rather than an invoice.

---

## Part E — Operations reality

### What one person can actually run

Be honest with yourself about this now, not during an incident.

| Responsibility | Hours/month | Sustainable solo? |
|---|---|---|
| Feature work | — | Yes, this is the point |
| Dependency updates | 2–4 | Yes, with Dependabot |
| Web Store review cycles | 2–3 | Yes |
| Support email | 3–6 | Up to ~500 users |
| Incident response | 1–4 | Only with a runbook |
| Security response (CVE) | 0–20 | Only because the attack surface is tiny |
| **Total operational** | **8–17** | **~40% of a part-time job** |

> **The honest number is about 40% of a part-time job, and it grows linearly with users while
> feature work does not.** This is the actual scaling limit of a solo local-first project, and
> it arrives at a few thousand users, not a few hundred thousand. **Knowing that now is what
> lets you choose the right answer later** — a paid tier, a support scope limit, or delegating
> ops — instead of discovering it during your worst week.

### The CVE response that is actually plausible

```bash
# Dependabot on. It is free and it does the boring half.
# But it does not triage for you, so:

# 1. Is it in a code path that ships?
pnpm why <vulnerable-package>

# 2. Is it reachable from a request handler?
grep -rn "from ['\"]<package>" apps/api/src/

# 3. Is the vulnerability actually exploitable for your use?
#    A path-traversal in a library you only use to parse YAML is
#    not the same as one you pass user input to.
```

```ts
// packages/audit.mjs — the answers, as a checklist, not a judgement call.
export const SEVERITY_MATRIX = {
  // In the extension, on a code path a page can reach → fix now.
  critical_reachable_extension: "patch within 24 hours, release extension",
  // Server-side, reachable from a request → patch within a week.
  high_reachable_api: "patch this week, add a regression test",
  // Present but unreachable → document why, revisit in 90 days.
  present_unreachable: "no action, add a note in §19",
  // Dev dependency only, not in any bundle → close the PR with a comment.
  dev_only: "no action",
}
```

> **The single most important security decision you already made is that there is no cloud LLM
> provider, no analytics, no ad SDK, and no third-party script on any page.** The typical web app
> has 40 transitive dependencies and inherits CVEs continuously. **Yours has ~15 and none of them
> execute on a page the user is visiting.** That is the whole reason your CVE load is an hour a
> month instead of a week.
>
> **It is also why the `onlyBuiltDependencies` list in `pnpm-workspace.yaml` should stay short.**
> Every build script you allow is code that runs with your credentials on install. Three entries
> is three; a list of twenty is a supply-chain posture you did not choose deliberately.

### Backup and recovery, concretely

```bash
# What exists, right now, for each kind of data:
#
#   Client vault (facts, docs)  → on the user's device. No backup. By design.
#     └─ Mitigation: the export feature (Ch.10). The user's responsibility.
#   MongoDB (ciphertext only)   → Atlas PITR, 7 days.
#     └─ Restores to the state 7 days ago. Clients re-pull and converge.
#   Code                        → git, and the remote is the backup.
#   Secrets                     → NOT in git. If you lose them, you must re-key.
```

```ts
// The disaster that actually happens: you lose the JWT private key.
export async function rotateSigningKey() {
  const nextKid = `ed25519-${new Date().toISOString().slice(0, 7)}`
  // 1. Generate the new key pair.
  // 2. Publish BOTH kids in /jwks.json for 24 hours.
  // 3. Sign with the new kid.
  // 4. After 24h, remove the old kid from JWKS.
  //
  // Clients cached the old public key. They keep verifying old
  // tokens until those 15-minute tokens expire. Nothing breaks.
  //
  // If you skipped `kid` in Chapter 14, this is a hard outage
  // and there is no graceful path. That is why kid was step one.
}
```

> **Your worst disaster is losing the JWT private key, and it is recoverable only because
> Chapter 14 made `kid` mandatory.** A key rotation without overlapping kids logs out every
> user at once. **Store the private key in two places.** Not one place with a good backup
> policy — two places, because the failure mode is "the one place has been silently failing for
> four months."

### What you did not build, and why that is correct

| Not built | Why this is the right call |
|---|---|
| Admin dashboard | You have 5 users. A `mongosh` is the same tool and takes one second |
| Redis | Chapter 12's rule: do not add a datastore to avoid adding a datastore |
| Read replicas | 1,000 users does not need one. It costs money and adds a replication-lag failure mode |
| Message queue | Sync is request/response. There is no work to defer |
| GraphQL | 12 endpoints, all REST. A resolver layer would be pure ceremony |
| Multi-region | Mumbai is correct. Mumbai is where your users are |
| Feature flags | You ship the extension to 100 people, not 100,000. A flag is a permanent tax for a temporary problem |
| Error tracking in the extension | Chapter 12's Sentry rule. Server-only, and it stays that way |

> **Every item on that list is a thing you could add. None of them is a thing you need.** The
> discipline of production hardening is not adding capability — it is **knowing which
> capabilities you are deliberately not building, and being able to say why.** That list is the
> answer to "why is this only one service?" and it is a better answer than any architecture
> diagram.

---

## Commit

```bash
git add -A
git commit -m "perf: bundle budgets, panel cold-start budget, content-script idle yield; a11y: axe gate, focus management, contrast fixes"
```

Add two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | PBKDF2 stays at 600k regardless of unlock latency | A vault that unlocks in 20ms has 6,000 iterations. Show a progress state instead of weakening the KDF. |
| 2026-10-XX | No Redis, no replicas, no queue, no admin dashboard | Every one is a permanent failure mode added before there is a problem it solves. A `mongosh` is the admin dashboard. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The three numbers: bundle 15MB, panel 400ms, scan 30ms** | **You.** Budgets are decisions |
| **Why 15MB is correct and "optimise" is not** | **You** |
| **The `host_permissions` surgical-list decision** | **You** |
| **`frame-ancestors 'none'` reasoning** | **You** |
| **The operations-hours table** | **You.** It changes what you build |
| **The `SEVERITY_MATRIX` triage** | **You** |
| Contrast ratio measurements | **You**, with the tool |
| The focus-management implementation | **You** — it is UX, and it is subtle |
| The screen reader walkthrough | **You.** Nobody can do this for you |
| `bundle-budget.sh` and CI gates | **OpenCode** |
| CSP header strings | **OpenCode** |
| The load tests | **OpenCode** — then read what each number *means* |
| `SECURITY.md` and the CVE template | **OpenCode** |
| The "did not build" table | **You** |

---

## Gotchas in this chapter

**The extension is 14MB and installs slowly.** You forgot `pdfjs-dist` and `tesseract` were
un-split into the initial chunk. `import()` them — see A2.

**The panel takes 900ms to open.** Import at module scope. Every heavy dependency must be behind
a dynamic `import()`.

**The panel is slow and it is your own PBKDF2.** Do not lower the iterations. Add a progress
state.

**Web pages feel 200ms slower with the extension enabled.** You are scanning synchronously in the
content script. `requestIdleCallback`, plus `setTimeout(0)` between frame batches.

**Profile read is 200ms.** Missing index on `key`. Run `explain()` and confirm an index scan.

**axe passes and a keyboard user still cannot use it.** Automation is 30%. Do B2 and B3 by hand.

**Focus vanishes to `<body>` when a review row is removed.** B3's unmount effect.

**VoiceOver says "button" forty times.** Missing `aria-label` on Chapter 9's icon buttons.

**Chips are 2.8:1.** `color-mix` at 14% against transparency. Mix against the surface instead.

**Your focus ring is invisible.** `*:focus { outline: none }` from the Chapter 3 reset, with no
replacement. Use `:focus-visible`.

**The mascot animates on the review screen.** Chapter 4's `prefers-reduced-motion` block is
targeted at the mascot. Add the blanket global rule.

**Chrome rejects the extension: "remote code."** You fetched Tesseract's WASM from a CDN. MV3
forbids it. Bundle it.

**Your content script has no `host_permissions` for the target site.** It works on your machine
because you have a persistent `chrome.scripting` override. Verify the shipped manifest.

**`localhost` is in the published `host_permissions`.** Remove it in the release workflow.

**A security review flags `JWT_PUBLIC_KEY` in the bundle.** It is public by design. Add an
allowlist entry with a comment, or someone will "fix" it and break offline verification.

**The API 413s on files under 10MB.** Your body limit is set to 1MB. Align it with the blob cap.

**A user hit the storage limit and never knew.** Show usage in Settings. Chapter 5's stub is
where it goes.

**You lost the JWT private key and every user is logged out.** You skipped `kid` in Chapter 14,
or stored the key in one place. See the rotation procedure.

**Atlas charged you $400.** You had no storage alert. M0 → M10 transition happened silently.

---

## Verify before moving on

- [ ] Bundle budgets enforced in CI and passing
- [ ] Panel interactive in under 400ms on a real page
- [ ] Vault unlock shows a progress state and PBKDF2 is still 600k
- [ ] Content script adds under 50ms to page FCP on a real portal
- [ ] Tesseract is not in the panel's initial chunk
- [ ] Profile read for 40 facts is under 15ms
- [ ] `axe --exit` passes on every web route
- [ ] Every interactive element is keyboard-reachable and activatable
- [ ] Focus moves somewhere real when a review row is removed
- [ ] Every icon button has an `aria-label` naming the action
- [ ] All body and chip text is 4.5:1 or better, verified with a tool
- [ ] Focus ring is visible and survives `:focus-visible`
- [ ] Reduced-motion emulation stops everything, including review transitions
- [ ] A screen reader walkthrough completes one full flow
- [ ] CSP set on the web app with `object-src 'none'` and `frame-ancestors 'none'`
- [ ] `host_permissions` lists specific form domains, no `<all_urls>`, no `localhost`
- [ ] No remotely hosted code in the extension
- [ ] Every API body has `.strict()` and a length limit
- [ ] 50 concurrent pushes produce 50 unique revisions
- [ ] A 500-fact pull returns metadata in under 400ms and under 60KB
- [ ] Storage usage is visible in Settings
- [ ] Atlas storage alert configured
- [ ] JWT private key stored in two places, with `kid` overlap tested
- [ ] You can name every item on the "did not build" list and say why

---

## Check yourself before Chapter 19

1. **Why is 15MB the right extension budget, and why is "optimise it" wrong?**
2. **What is the difference between bundled and loaded, and which one matters for panel speed?**
3. **Why must PBKDF2 stay at 600k even when unlock feels slow?**
4. **What does `requestIdleCallback` do in the content script, and what happens without it?**
5. **Why does `color-mix` at 14% fail contrast, and what do you mix against instead?**
6. **What does removing a review row without a focus effect break?**
7. **Why does `frame-ancestors 'none'` matter more here than for a typical app?**
8. **Why is a surgical `host_permissions` list better for review *and* for trust?**
9. **What does `kid` in the JWT protect you from, and what happens without it?**
10. **Which item on the "did not build" list would you add first, and at what trigger?**

---

**Next: [Chapter 19 — Ship Checklist](./19-ship-checklist.md)** — the final gate. Phase 1 exit
criteria, the Web Store submission, the privacy verification, and the honest list of what this
product does not do.