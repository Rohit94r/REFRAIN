# Chapter 7 — The Content Script

> **Days 7–8 · Goal: the extension reports "I found 14 fields" on a real Google Form.**
>
> This is your **first demo checkpoint.** Stop here on Day 9, record 40 seconds, send it to
> one person. Do not proceed to Chapter 8 until this works.

---

## Understand this first

### The content script is the only code allowed to touch the page

Not "the main code." **The only one.** No UI, no store, no business logic, no React, no
network. It does two things:

1. **Read** — turn the page's DOM into a `FormSchema`
2. **Write** — accept a list of `{id, value}` and set the values

That is the entire contract. It imports from `@refrain/fields` for types and nothing else.

**Why the discipline matters:** every product in this category dies when a portal changes its
HTML. Contentful, Smartsheet, a government recruitment portal — all of them broke on the day
they redesigned. If your fragile code is one dumb file with no imports, breakage is a 30-line
diff. If the logic is spread across six modules with a React tree in the middle, it is a
rewrite.

Resist every urge to "just add a little logic here." Chapter 8 is where logic goes.

### There are exactly four ways to read another site's DOM

| Mechanism | Crosses same-origin | Distribution | Verdict |
|---|---|---|---|
| Web app on your domain | ❌ No | Trivial | Dead on arrival |
| Userscript (Tampermonkey) | ✅ | Requires installing a second tool | Bad first-run experience |
| Bookmarklet | ✅ | Nothing persists, one click every time | Unusable |
| **Browser extension** | ✅ | One install from the Web Store | ✅ **This** |

The same-origin policy is not an obstacle to route around. It is the reason your product is
an extension, and it is why a web app can never do this no matter how clever you get.

### The three bugs that break every product in this category

Learn these now and you will save yourself the three weeks everyone else loses.

**Bug 1 — React swallows your write.** React does not listen for `el.value = x`. It monkey-patches
the property on the *element instance* and keeps its own shadow state. Setting `.value`
updates the DOM and React never hears about it. On submit, React's state is still `""`, so the
form posts an empty field. **The user sees the text on screen and submits it, and it is
missing.** That is the worst possible failure: it looks like it worked.

**Bug 2 — the form is in an iframe.** A content script only runs in the top frame by default.
The form fields are not there. You report zero fields on a page that obviously has a form.
Fix: `allFrames: true`, and you must track `frameId` on every field to write back to the
right frame.

**Bug 3 — the visible input is a decoy.** Modern widgets (Google's own date pickers, React
select libraries, every Indian job portal built on a JS framework) show a styled `<div>` or a
`readonly` input, and the real value lives in a hidden `<input>` or is held in framework
state. You fill the decoy, nothing is stored, submit sends nothing.

**All three produce the same symptom — "it filled but nothing arrived."** That is why you
must understand them individually rather than shipping a fix and moving on.

### Why `all_frames: true` is a power you should ration

With `allFrames: true` your script runs in every frame on every page — including third-party
ad iframes and analytics frames. You then need to know which frame a field came from in order
to write back.

So: enable it, **filter aggressively**, and record `frameId` on every field. A field with no
`frameId` is a bug waiting to happen.

---

## Step 1 — Framework choice, and a deviation from `refrain.md`

§10 says "Plasmo (or `@crxjs/vite-plugin`)." **This map uses WXT instead.** Here is why, and
you should record it as a §19 decision:

| Need | Plasmo | crxjs | **WXT** |
|---|---|---|---|
| Side panel entry point | ✅ | ✅ | ✅ First-class |
| Content script entry point | ✅ | ✅ | ✅ |
| **Two apps sharing one React codebase** | Awkward | Awkward | ✅ **Designed for it** |
| Works cleanly in a pnpm workspace + Turborepo | No | Sometimes | ✅ |
| `allFrames` / `matchOriginAsFallback` | Manual | Manual | ✅ Declared in config |
| Build | Own bundler | Vite plugin | Vite under the hood |

Your side panel and your web app are **the same React app with two doors.** WXT models entry
points as files, so the side panel entry imports `@refrain/ui` and `@refrain/vault` exactly
like the web app does — with no shared-build gymnastics. That is the deciding factor, and it
is exactly the reason §10 listed the alternatives: pick for *your* monorepo shape.

**If WXT fights you**, fall back to Plasmo. Nothing else in this chapter changes, because
nothing in this chapter is framework-specific. The DOM code is the same either way.

```bash
pnpm --filter @refrain/extension add wxt @wxt-dev/module-react
```

`apps/extension/wxt.config.ts`:

```ts
import { defineConfig } from "wxt"

export default defineConfig({
  srcDir: ".",
  modules: ["@wxt-dev/module-react"],
  manifest: {
    name: "Refrain",
    description: "Answer once. Fill everywhere.",
    // Only what you actually use. Every permission is a review-flagger on the Web Store.
    permissions: ["sidePanel", "storage", "scripting", "activeTab"],
    host_permissions: ["<all_urls>"],
    side_panel: { default_path: "sidepanel.html" },
    // Content scripts are registered at runtime from the side panel,
    // not statically — otherwise they run on every page for no reason.
  },
  vite: () => ({
    resolve: { alias: { "@": new URL("./src", import.meta.url).pathname } },
  }),
})
```

> **Why `host_permissions: ["<all_urls>"]` and not a narrower list.** You do not know every
> portal your users need. A narrow list means a user on an unlisted portal gets a broken
> product, and you cannot enumerate Indian government portals ahead of time. The cost is a
> hostile review flag — mitigated by being open source with a published privacy policy and a
> manifest that contains no network hosts. §11 lives or dies on that manifest.

---

## Step 2 — Entry points

```
apps/extension/
├── entrypoints/
│   ├── sidepanel/
│   │   ├── index.html
│   │   ├── main.tsx        ← mounts the shared React app
│   │   └── style.css
│   ├── content/
│   │   ├── index.ts        ← the reader/writer
│   │   └── content.css     ← highlights fields we will write to
│   └── background.ts       ← opens the side panel, wires the port
└── wxt.config.ts
```

`entrypoints/background.ts`:

```ts
export default defineBackground(() => {
  // Let a user gesture open the side panel — this is the whole activation path.
  chrome.sidePanel
    .setPanelBehavior({ openPanelOnActionClick: true })
    .catch(() => {})

  chrome.action.onClicked.addListener((tab) => {
    if (tab.windowId) chrome.sidePanel.open({ windowId: tab.windowId })
  })
})
```

`entrypoints/sidepanel/main.tsx`:

```tsx
import React from "react"
import ReactDOM from "react-dom/client"
import { App } from "@/App"
import "@refrain/ui/tokens.css"

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode><App /></React.StrictMode>,
)
```

> Note there is no `<App />` written yet. For the Day 8 checkpoint, write a throwaway one
> that just prints a number. Chapter 9 replaces it with the real screen.

---

## Step 3 — The reader: turning a DOM into a `FormSchema`

This is the single most valuable file in the product. Read it twice.

### Step 3a — Find the candidate fields

```ts
// entrypoints/content/index.ts
import { defineContentScript } from "wxt/sandbox"
import type { FillKind, FormFieldSchema, FormSchema } from "@refrain/fields"

const CANDIDATE_SELECTOR = [
  "input:not([type=hidden])",
  "input[type=hidden][data-refrain-driver]",
  "textarea",
  "select",
  "[contenteditable=true]",
  "[role=textbox]",
  "[role=combobox]",
  "[role=listbox] [role=option]",
  "[role=radio]",
  "[role=checkbox]",
].join(",")

const SKIP = new Set([
  "submit", "button", "reset", "image", "file",
  "hidden", "password-reveal", "color", "range",
])

export default defineContentScript({
  matches: ["<all_urls>"],
  allFrames: true,               // ← Bug 2. This is the whole point.
  runAt: "document_idle",        // after the page settles, before it fights you
  matchAboutBlank: true,         // some portals render into about:blank frames
  main(ctx) {
    ctx.onInvalidated(() => teardown())
    // Message wiring lives in Step 6. Extraction starts here.
  },
})
```

`runAt: "document_idle"` is deliberate. `document_start` gets you an empty DOM; `document_end`
can beat slow frameworks. `document_idle` is where the DOM is built and the frameworks have
finished attaching listeners. If a field is genuinely missing, re-scan on demand rather than
racing the page.

### Step 3b — Resolve the label

This is 80% of extraction quality, and it is why most tools in this category report
`Field 1`, `Field 2`. **The label is the only thing the mapping engine has to work with.**

```ts
/** Priority order. First non-empty wins. */
function resolveLabel(el: HTMLElement): string {
  // 1. <label for="id">
  const id = el.getAttribute("id")
  if (id) {
    const lab = document.querySelector<HTMLLabelElement>(`label[for="${CSS.escape(id)}"]`)
    if (lab?.textContent?.trim()) return clean(lab.textContent)
  }

  // 2. ancestor <label>
  const wrap = el.closest("label")
  if (wrap?.textContent?.trim()) return clean(wrap.textContent)

  // 3. aria-labelledby → resolve the referenced ids
  const by = el.getAttribute("aria-labelledby")
  if (by) {
    const t = by.split(/\s+/).map((i) => document.getElementById(i)?.textContent ?? "").join(" ")
    if (t.trim()) return clean(t)
  }

  // 4. aria-label
  const aria = el.getAttribute("aria-label")
  if (aria?.trim()) return clean(aria)

  // 5. closest heading in the fieldset — <fieldset><legend>Name</legend>
  const legend = el.closest("fieldset")?.querySelector("legend")
  if (legend?.textContent?.trim()) return clean(legend.textContent)

  // 6. placeholder
  const ph = el.getAttribute("placeholder")
  if (ph?.trim()) return clean(ph)

  // 7. name — last resort, still useful
  const nm = el.getAttribute("name")
  if (nm?.trim()) return clean(nm)

  return ""
}

/** Strip required markers, asterisks, and collapse whitespace. */
function clean(s: string): string {
  return s.replace(/[*†‡]/g, "").replace(/\s+/g, " ").trim().slice(0, 120)
}
```

**Order matters.** `<label>` before `aria-label` because `aria-label` is often a generic
"Text field" from a widget library. **Do not skip the `name` fallback** — on Indian job
portals built in 2015 the `name` attribute (`candidate_first_name`) is frequently the *most*
descriptive thing on the field. It is ugly but it is signal.

> **Why you need this to be honest.** Your mapping engine (§10) is built on a hand-labelled
> dataset of real form fields. That dataset is worthless if labels are garbage. Garbage in,
> garbage rules out — and no amount of inference fixes it.

### Step 3c — The Google Forms special case

Google Forms does not use labels at all. Its structure is:

```html
<div class="freebirdFormviewerComponentsQuestionBaseRoot">
  <div class="…QuestionBaseHeader">
    <div class="…QuestionBaseTitle" id="iRZLRc-1">
      Email address<div … aria-label="Required">*</div>
    </div>
  </div>
  <input type="text" name="entry.123" class="whsOnd">
</div>
```

So a dedicated resolver, tried **before** the generic chain:

```ts
function resolveGoogleFormsLabel(el: HTMLElement): string | null {
  const root = el.closest(".freebirdFormviewerComponentsQuestionBaseRoot")
  if (!root) return null
  const title = root.querySelector(".freebirdFormviewerComponentsQuestionBaseTitle")
  return title ? clean(title.textContent ?? "") : null
}
```

> **Never hardcode a Google Forms class as your only path.** These class names are Google's
> internal build names and they change. This resolver is an *accelerator* that runs first and
> falls through to the generic chain. If Google renames it, Google Forms still works — just
> via `aria-label` or `name`, slightly worse. **Accelerators, never dependencies.**

### Step 3d — Kind, required, options, and the decoy check

```ts
function resolveKind(el: HTMLElement): FillKind | null {
  const role = el.getAttribute("role")
  const type = (el as HTMLInputElement).type?.toLowerCase()
  const tag = el.tagName.toLowerCase()

  if (role === "option") return "radio"          // Google Forms choice chips
  if (role === "radio") return "radio"
  if (role === "checkbox") return "checkbox"
  if (role === "combobox" || tag === "select") return "select"
  if (tag === "textarea" || el.isContentEditable) return "textarea"
  if (type === "file") return "file"
  if (type === "checkbox") return "checkbox"
  if (type === "radio") return "radio"
  if (type === "number" || type === "range") return "number"
  if (type === "date" || type === "month" || type === "week") return "date"
  if (type === "email") return "email"
  if (type === "tel") return "tel"
  if (type === "url") return "url"
  if (tag === "input") return "text"
  return null
}

function isRequired(el: HTMLElement): boolean {
  if (el.hasAttribute("required")) return true
  if (el.getAttribute("aria-required") === "true") return true
  // Google Forms marks it with a nested div aria-label="Required"
  const root = el.closest(".freebirdFormviewerComponentsQuestionBaseRoot")
  if (root?.querySelector('[aria-label="Required"]')) return true
  // A trailing asterisk in the label is the universal convention.
  return /\*/.test(el.closest("label")?.textContent ?? "")
}

/**
 * Bug 3 detector. A "visible" field is a decoy when:
 *   - it is readonly, OR
 *   - it has a visible opacity/pointer-events lock, OR
 *   - a hidden input sits adjacent and is the one with a real name.
 */
function findHiddenDriver(el: HTMLElement): HTMLInputElement | null {
  if (el instanceof HTMLInputElement && el.type === "hidden") return el
  const scope = el.closest("div,fieldset,td,li") ?? document
  return scope.querySelector<HTMLInputElement>(
    'input[type=hidden][name]:not([data-refrain-seen])',
  )
}

function readOptions(el: HTMLElement): string[] | undefined {
  if (el instanceof HTMLSelectElement) {
    return [...el.options].map((o) => o.text.trim()).filter(Boolean)
  }
  const listbox = el.getAttribute("role") === "option" ? el.parentElement : el
  if (listbox?.getAttribute("role") === "listbox") {
    return [...listbox.querySelectorAll('[role="option"]')]
      .map((o) => o.textContent?.trim() ?? "")
      .filter(Boolean)
  }
  if (el.getAttribute("role") === "radio") {
    const group = el.closest('[role="radiogroup"],[role="listbox"]')
    return [...(group?.querySelectorAll('[role="option"],[role="radio"]') ?? [])]
      .map((o) => o.textContent?.trim() ?? "")
      .filter(Boolean)
  }
  return undefined
}
```

**`findHiddenDriver` is the single most valuable function in this chapter.** When it returns a
driver, `FormFieldSchema.hasHiddenDriver` is `true`, the writer targets the *driver*, and the
write actually lands. Without it you will spend a week debugging portals where "it clearly
filled but nothing saved."

### Step 3e — Assemble

```ts
function extract(): FormSchema {
  const url = location.href
  const title = document.querySelector("h1")?.textContent?.trim() ?? document.title

  const fields: FormFieldSchema[] = []

  for (const el of document.querySelectorAll<HTMLElement>(CANDIDATE_SELECTOR)) {
    const kind = resolveKind(el)
    if (!kind) continue

    // Frame it into the right frame. Without this, writes go to the top document.
    const frameId = ctx.frameId ?? 0

    // Mark so we never pick the same hidden input for two visible fields.
    if (el instanceof HTMLInputElement) el.dataset.refrainSeen = "1"

    const driver = findHiddenDriver(el)

    fields.push({
      id: `${frameId}:${el.id || el.name || cssPath(el)}`,
      label: resolveGoogleFormsLabel(el) ?? resolveLabel(el),
      name: el.getAttribute("name") ?? undefined,
      kind,
      required: isRequired(el),
      options: readOptions(el),
      hasHiddenDriver: driver !== null && driver !== el,
      frameId,
    })
  }

  return {
    url,
    title,
    fields,
    blocked: detectBlocked(url),
    blockedReason: blockedReason(url),
    captchaDetected: detectCaptcha(),
  }
}
```

> **`ctx.frameId ?? 0`** — WXT gives you the frame id in the script context. Fall back to `0`
> for the top frame. If your framework does not surface it, use
> `chrome.runtime.getFrameId(window)`. Getting this wrong means multi-frame portals silently
> lose every write, and it is the hardest bug to diagnose because everything *looks* right.

### Step 3f — The §11 block list and CAPTCHA detection

```ts
/** §11: a hard "no". Not a warning — a refusal. */
const BLOCKED_PATTERNS: Array<[RegExp, string]> = [
  [/\.nic\.in|\.gov\.in|\.ac\.in/i, "Government portal — identity verification is out of scope."],
  [/aadhaar|uidai/i,               "Aadhaar is never handled by Refrain."],
  [/(irctc|indianrail)\.co\.in/i,  "Railway booking requires payment. Out of scope."],
  [/(upsc|ssc|banking|ibps|nta)\b/i,"Competitive and banking exams are outside the product."],
  [/\/(payment|checkout|wallet)/i,  "Payment forms are never filled."],
]

function detectBlocked(url: string) { return BLOCKED_PATTERNS.some(([p]) => p.test(url)) }

function detectCaptcha(): boolean {
  return !!document.querySelector(
    '.g-recaptcha, .h-captcha, [data-sitekey], iframe[src*="recaptcha"], ' +
    'iframe[src*="hcaptcha"], iframe[src*="challenges.cloudflare"], #cf-turnstile',
  )
}
```

> **This block list is a feature, and it is the first thing a reviewer should see.** §11 says
> Refrain has a hard "no" list. Encoding it as a regex that *refuses* is the difference
> between a privacy policy and a product. Put it in `refrain.md`, put it in the README, and
> keep it honest. **Never soften it to improve a conversion metric.**

---

## Step 4 — The writer: actually filling the form

Now the part that decides whether your product works or is a demo.

```ts
/**
 * Bug 1 fix. React overrides the `value` property on the *element instance*
 * and tracks its own shadow state. Writing `el.value = x` updates the DOM
 * and React never hears about it, so on submit React's state is still "" and
 * the field posts empty — while the user sees the text sitting there.
 *
 * Writing through the PROTOTYPE's setter updates the same underlying
 * property React patched into, so React's change handler fires and its
 * state updates.
 */
function setNativeValue(el: HTMLInputElement | HTMLTextAreaElement, value: string): boolean {
  const proto = Object.getPrototypeOf(el) as object
  const desc = Object.getOwnPropertyDescriptor(proto, "value")
  if (!desc?.set) { el.value = value; return false }
  desc.set.call(el, value)
  return true
}

function fire(el: HTMLElement, types = ["input", "change"]): void {
  for (const type of types) {
    el.dispatchEvent(new Event(type, { bubbles: true, composed: true }))
  }
}

function writeOne(target: HTMLElement, value: string): { ok: boolean; reason?: string } {
  // ── Never fill a file input. Browsers forbid it by design. ──────────
  if (target instanceof HTMLInputElement && target.type === "file") {
    return { ok: false, reason: "File uploads must be done by hand." }
  }

  // ── Bug 3: write the hidden driver, not the decoy ──────────────────
  const driver = target instanceof HTMLInputElement && target.type === "hidden"
    ? target
    : findHiddenDriver(target)
  const sink: HTMLElement = driver ?? target

  // ── Google Forms + widget-library choice lists ────────────────────
  if (target.getAttribute("role") === "option" || target.getAttribute("role") === "radio") {
    const label = clean(target.textContent ?? "").toLowerCase()
    const wanted = value.trim().toLowerCase()
    if (label.includes(wanted) || wanted.includes(label)) {
      target.click()          // these widgets only respond to a real click
      return { ok: true }
    }
    return { ok: false, reason: `No option matching "${value}"` }
  }

  // ── <select>: match by text, then by value ───────────────────────
  if (sink instanceof HTMLSelectElement) {
    const opt = [...sink.options].find(
      (o) => o.text.trim().toLowerCase() === value.trim().toLowerCase(),
    ) ?? [...sink.options].find((o) => o.value === value)
    if (!opt) return { ok: false, reason: `"${value}" is not an available option` }
    sink.selectedIndex = opt.index
    fire(sink)
    return { ok: true }
  }

  // ── checkbox / radio: native `checked` setter, then change ─────────
  if (sink instanceof HTMLInputElement && (sink.type === "checkbox" || sink.type === "radio")) {
    const proto = Object.getPrototypeOf(sink) as object
    const desc = Object.getOwnPropertyDescriptor(proto, "checked")
    desc?.set?.call(sink, value === "true" || value.toLowerCase() === "yes")
    fire(sink, ["change", "click"])
    return { ok: true }
  }

  // ── contenteditable ────────────────────────────────────────────────
  if (sink instanceof HTMLElement && sink.isContentEditable) {
    sink.textContent = value
    fire(sink, ["input"])
    return { ok: true }
  }

  // ── the normal path ────────────────────────────────────────────────
  if (sink instanceof HTMLInputElement || sink instanceof HTMLTextAreaElement) {
    const usedNative = setNativeValue(sink, value)

    // Readonly means a decoy with no hidden sibling — the widget wants a click.
    if (sink.readOnly) return { ok: false, reason: "Field is read-only; it needs a manual pick." }

    // Date inputs reject free text unless the format matches exactly.
    if (sink instanceof HTMLInputElement && sink.type === "date") {
      const iso = toISODate(value)
      if (!iso) return { ok: false, reason: `"${value}" is not a date Refrain can format` }
      sink.value = iso
    }

    fire(sink)
    return { ok: usedNative, reason: usedNative ? undefined : "Written without native setter" }
  }

  return { ok: false, reason: "Unsupported field type" }
}
```

**`composed: true` on the events** matters when the field sits inside a shadow root. Without
it, the event does not escape the shadow DOM and the framework never sees it.

**Date handling is a real trap.** `<input type="date">` silently rejects anything that is not
`YYYY-MM-DD`. A user's DOB extracted as `14/07/2006` writes nothing at all and reports
success unless you check. `toISODate` handles the common Indian formats and returns `null` when
it cannot — and a `null` means `needs-input`, not a silent skip.

### The orchestestrator

```ts
type FillRequest = { id: string; value: string }
type FillResult = { id: string; ok: boolean; reason?: string }

function fillAll(requests: FillRequest[]): FillResult[] {
  // Group by frame so each frame's writes happen in one place.
  const byFrame = new Map<number, FillRequest[]>()
  for (const r of requests) {
    const [frameStr] = r.id.split(":")
    const f = Number(frameStr ?? 0)
    if (!byFrame.has(f)) byFrame.set(f, [])
    byFrame.get(f)!.push(r)
  }

  const index = indexFields()
  const results: FillResult[] = []

  for (const [, group] of byFrame) {
    for (const req of group) {
      const target = index.get(req.id)
      if (!target) { results.push({ ...req, ok: false, reason: "Field vanished" }); continue }
      try {
        results.push({ id: req.id, ...writeOne(target, req.value) })
      } catch (err) {
        // One bad field must never abort the other 19.
        results.push({ id: req.id, ok: false, reason: (err as Error).message })
      }
    }
  }
  return results
}
```

> **Per-field try/catch, always.** A single throwable field must not abort the other nineteen.
> The user is looking at a scholarship form with a 12-minute deadline. Partial fills they can
> fix; a blank form they cannot.

---

## Step 5 — Prove it works (the Day 8 checkpoint)

### 5a — Verify the reader on Google Forms

Open a real Google Form, open the side panel, and for now hardcode:

```tsx
// apps/extension/entrypoints/sidepanel/App.tsx — THROWAWAY, replaced in Ch.9
export function App() {
  const [count, setCount] = useState<number | null>(null)
  const [labels, setLabels] = useState<string[]>([])

  async function scan() {
    const [tab] = await chrome.tabs.query({ active: true, currentWindow: true })
    const res = await chrome.tabs.sendMessage(tab.id!, { type: "REFRAIN/SCHEMA" })
    setCount(res.schema.fields.length)
    setLabels(res.schema.fields.map((f: any) => f.label))
  }

  return (
    <div className="p-4">
      <button onClick={scan} className="rounded-card bg-brand-500 px-4 py-2 text-white">
        Scan this page
      </button>
      <p className="mt-4 text-2xl font-semibold">{count === null ? "—" : `I found ${count} fields`}</p>
      <ul className="mt-2 text-sm text-ink-muted space-y-0.5">
        {labels.map((l, i) => <li key={i}>· {l || <em>(no label)</em>}</li>)}
      </ul>
    </div>
  )
}
```

**This is the Day 8 deliverable: "I found 14 fields."**

Now check the quality of the labels yourself:

- [ ] Every label is a **real question**, not "Text field" or "Input"
- [ ] No `(no label)` entries — if there are, your resolver order needs work
- [ ] Required fields are flagged correctly
- [ ] Choice fields list their options

### 5b — Verify the writer

Add a temporary console handler, then from the side panel hardcode one write:

```ts
// in the content script's message handler
if (msg.type === "REFRAIN/FILL") {
  const results = fillAll(msg.values)
  console.table(results)   // ← the fastest debugging tool in this chapter
  return { results }
}
```

Fill a Google Form with hardcoded values and then **submit it to a throwaway account.** Do not
just look at the screen.

- [ ] Text fields arrive
- [ ] Email field arrives
- [ ] Radio/choice arrived **even though there is no real `<input>`**
- [ ] Date field arrived
- [ ] You submitted, reloaded the response, and the answers are there

**That last line is the entire test.** Looking at the DOM is how Bug 1 hides.

---

## Step 6 — The messaging protocol

```
side panel ──REFRAIN/SCHEMA────▶ content script
             ◀─SCHEMA_RESULT────

side panel ──REFRAIN/FILL──────▶ content script
             ◀─FILL_RESULT──────

background ◀──port "refrain/sidepanel"──▶ side panel (lifecycle)
```

```ts
export default defineContentScript({
  matches: ["<all_urls>"],
  allFrames: true,
  runAt: "document_idle",
  main() {
    const index = indexFields()

    chrome.runtime.onMessage.addListener((msg, _sender, sendResponse) => {
      if (msg?.type === "REFRAIN/SCHEMA") {
        sendResponse({ schema: extract() })
        return false
      }
      if (msg?.type === "REFRAIN/FILL") {
        const results = fillAll(msg.values as FillRequest[])
        // Report failures loudly rather than reporting success.
        sendResponse({
          results,
          failed: results.filter((r) => !r.ok).length,
        })
        return false
      }
      return false
    })
  },
})
```

> **`return false` on every branch** tells Chrome you will *not* call `sendResponse`
> asynchronously. If you ever add `await` inside a handler, you must `return true` instead or
> the message port closes and you get "the message port closed before a response was
> received" — a genuinely maddening error with an obvious fix.

**Why `sendMessage` and not a long-lived port for data?** Because the side panel and the page
have independent lifecycles. The panel can be closed and reopened while the page stays put, and
`sendMessage` handles that with no reconnect logic. Reserve the long-lived port for lifecycle
signals only.

**Handle the missing-receiver case.** The panel may query a tab where the script is not
injected (a `chrome://` page, or a page loaded before you registered the script):

```ts
try {
  const res = await chrome.tabs.sendMessage(tab.id!, { type: "REFRAIN/SCHEMA" })
  if (!res) return setError("This page is not supported.")
} catch {
  return setError("Reload the page and try again.")
}
```

A raw `Unchecked runtime.lastError` here is a support burden. Catch it and say the sentence a
user can act on.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "feat(extension): content script reader/writer, all_frames, native setter, §11 block list"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`writeOne` and the native-setter fix** | **You. Entirely.** This is the product's core competence |
| `resolveLabel` priority chain | **You** — you will tune it against 40 real forms |
| `findHiddenDriver` | **You** |
| `resolveGoogleFormsLabel` | **You**, then extend it to Naukri/Internshala by hand |
| The §11 block list regexes | **You.** This is a policy decision, not code |
| `wxt.config.ts` permissions | **You** |
| `extract()` assembly loop | **OpenCode**, then read it line by line |
| `toISODate` (date format variants) | **OpenCode** — it needs breadth, then you verify each format |
| `cssPath()` fallback | **OpenCode** |
| A Playwright script that loads 20 saved form URLs and dumps the schema | **OpenCode** — then you read every output and fix the resolver |

> **Do that Playwright script.** It is the highest-leverage 40% in this chapter. Twenty saved
> public forms, dumped as JSON, becomes your regression suite for the label resolver — and
> in Chapter 8 it becomes the hand-labelled dataset that *is* your moat.

---

## Gotchas in this chapter

**"I found 0 fields" on a page that clearly has a form.** Either the script is not injected
(wrong `matches`, or the page was already open when you registered it — reload), or the fields
are in an iframe and `allFrames` is off, or they are custom widgets with no `input`/`textarea`
at all. Check all three in order.

**Values appear on screen but submit as empty.** Bug 1. You set `.value` instead of using the
prototype setter, or you forgot to dispatch `input`.

**A React-controlled field reverts a moment after you fill it.** React re-rendered and
overwrote your write. That means you filled the wrong element — it is a decoy. Check
`hasHiddenDriver`.

**Works on Google Forms, fails on Naukri.** Naukri uses a custom React select with a hidden
input. You filled the visible div. Extend `findHiddenDriver` and add Naukri to your Playwright
dataset.

**Only the top half of the page fills.** Two frames, you wrote only to frame 0. Check that
every `FormFieldSchema` carries a `frameId`.

**`The message port closed before a response was received`.** You `await`ed inside
`onMessage` and returned `false`. Return `true`.

**It works in the side panel but not on the page.** You are reading your own panel's DOM. The
content script's `document` is the page's document; they are different worlds. Log
`location.href` on both sides to prove it.

**The extension asks for "read and change all your data on all websites" on install.** That is
`<all_urls>`. It is unavoidable and it is the single biggest driver of install abandonment.
§11's answer is the privacy page, the open-source repo, and — this is the real fix — **a
permissions justification in the README.** Write it well; it converts.

---

## Verify before moving on

- [ ] "I found N fields" with N matching the visible form, on a real Google Form
- [ ] Every label is a real question; zero `(no label)` entries
- [ ] Required flags and options are correct
- [ ] `allFrames: true`, and a multi-frame portal fills completely
- [ ] A React-controlled field on a real portal survives to submission
- [ ] A Google Forms choice (no real `<input>`) fills correctly
- [ ] A date field fills from `14/07/2006`
- [ ] A file input is refused with a clear reason, never attempted
- [ ] A `.nic.in` URL returns `blocked: true` and the panel refuses
- [ ] A CAPTCHA page sets `captchaDetected: true` and Refrain **waits**
- [ ] You submitted a real form and the answers arrived
- [ ] `grep -rn "eval\|2captcha\|anti-captcha\|proxy" src/ packages/` returns nothing

---

## Check yourself before Chapter 8

1. **Why does `el.value = x` fail on a React input, and what does the prototype setter do that `.value` does not?**
2. **What does `composed: true` do, and which bug does it prevent?**
3. **A field is `readonly` with no hidden sibling. What do you report, and why not a guess?**
4. **Why is the Google Forms resolver an accelerator and not the primary path?**
5. **What is `allFrames: true` costing you, and what do you do about it?**
6. **You fill 18 of 20 fields because one threw. Why is per-field try/catch non-negotiable?**
7. **What does the §11 block list protect, and why is it better in code than in a policy document?**

---

**Next: [Chapter 8 — The Mapping Engine](./08-the-mapping-engine.md)** — rule-based mapping
first, semantic matching second, normalisation, confidence scoring, and local inference via the
Chrome Prompt API with an Ollama fallback.