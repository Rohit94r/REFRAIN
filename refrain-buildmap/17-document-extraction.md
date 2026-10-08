# Chapter 17 — Document Extraction

> **Day 17 · Goal: a text PDF and a photographed marksheet both produce reviewable facts with
> verifiable provenance.**
>
> The least glamorous chapter and the one that decides whether Refrain saves the user ten
> minutes or thirty. It is also the chapter where a naive implementation quietly corrupts data.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **Extraction** — getting facts out of a document you uploaded.
- **PDF** — a document format that stores text and images.
- **PDF.js** — the library that reads a PDF in the browser.
- **OCR** — turning a picture of text into real, selectable text.
- **Tesseract** — the OCR engine used here. It can run inside the browser.
- **Scanned PDF** — a PDF that is really just pictures. Needs OCR.
- **Text layer** — the real text in a normal PDF. No OCR needed.
- **Confidence** — for OCR, how sure it is of what it read.
- **Locator** — where a value came from: page number and matched line.
- **Promotion** — when a value you confirmed becomes fully trusted.
- **Web Worker** — a background thread, so heavy work does not freeze the page.

---

## Understand this first

### Two completely different problems wearing one UI

**Digitally-created PDFs** — most modern government documents, bank statements, transcripts.
There is a text layer. `pdf.js` reads it in milliseconds at 99% accuracy. Confidence 0.95,
auto-fills, done.

**Photographed marksheets** — which is most Indian marksheets. There is no text layer at all,
just a picture of paper. You need OCR, and OCR on a phone photo of a folded, creased,
low-light marksheet is genuinely bad.

That gap defines the entire feature, and it means the whole design question is: **how do you
ship OCR without ever putting a wrong value into a form?**

### The answer is a confidence number, and it is structural

| Source | Confidence | Against `REVIEW_THRESHOLD` (0.75) | Behaviour |
|---|---|---|---|
| pdf.js text layer | **0.95** | Above | Fills automatically. Green chip. |
| OCR (Tesseract) | **0.62** | Below | **Never auto-fills.** Amber chip. User confirms. |
| User-confirmed | **1.0** `manual` | Above | Fills forever, everywhere |

This is the whole design. It is not a tuning knob and you should not move it.

Consider what OCR might return for a name: `Rohit Jadhav`, `Rohlth Jadhav`, `Rohit | Jadhav`,
`Rohit Jadha`, `ROHIT JADHAV`. At 0.62 that value **never reaches a form by itself.** It sits
in your vault waiting for the user to confirm it — one tap, two seconds — and then it is
`manual` provenance at 1.0 permanently, and **every future form fills correctly because of
that one tap.**

> **That is what makes OCR safe to ship at all.** Not accuracy. Not a better model. One
> confirmation per fact, once, converts a permanently-untrusted source into a permanently-trusted
> one. A user's single tap buys unlimited future correctness.

### `locator` is not metadata — it is the conversion mechanism

```
from marksheet.pdf · p.3
```

The user opens the PDF, flips to page 3, sees the name, and confirms. **Three seconds.**

Without a locator they must decide whether to trust an extraction from a file they may not
remember uploading. That is a completely different decision, and most people make it by
refusing to use the feature at all.

**Provenance is not a nice thing to display. It is the mechanism that converts untrusted
automation into trusted data.** Build it as a first-class field, not as a tooltip.

### Never `JSON.stringify` a PDF

This one will corrupt data and OOM your panel, and it is an easy mistake because
`seal()` in Chapter 12 does `JSON.stringify` internally — which is *correct* for your profile
objects and *catastrophic* for a 4MB binary blob.

You need a dedicated byte-sealing path. Add it now:

```ts
// packages/vault/src/crypto/aead.ts — ADD
export async function sealBytes(key: CryptoKey, bytes: ArrayBuffer): Promise<Sealed> {
  const iv = crypto.getRandomValues(new Uint8Array(12))
  const ciphertext = await crypto.subtle.encrypt(
    { name: "AES-GCM", iv: iv as BufferSource },
    key,
    bytes,                      // ← no stringify. The bytes go in as bytes.
  )
  return { ciphertext, iv }
}

export async function unsealBytes(key: CryptoKey, sealed: Sealed): Promise<ArrayBuffer> {
  return crypto.subtle.decrypt(
    { name: "AES-GCM", iv: sealed.iv as BufferSource },
    key,
    sealed.ciphertext,
  )
}
```

---

## Step 1 — The pipeline

```
  file
    ↓
  is it a PDF?
    ├─ no  → image path (createImageBitmap + EXIF orientation) ─┐
    ↓                                                            │
  pdf.js → text layer per page                                   │
    ↓                                                            │
  text.length < 80 on this page?                                 │
    ├─ yes → SCAN. Rasterise at 2× → Tesseract → 0.62 ──────────┤
    └─ no  → 0.95                                               │
    ↓                                                            ↓
  extractFacts(pages, filename) ← rules + locator + provenance
    ↓
  CONFIRM GATE  (every fact below threshold requires a tap)
    ↓
  vault, as `manual` provenance at 1.0
```

**The branch on `text.length` is the whole routing decision.** One number decides which engine
runs, and therefore which confidence applies. Do not try to decide by file size, file name, or
file size heuristics — a 400KB text PDF and a 400KB scan are both normal.

---

## Step 2 — pdf.js, correctly

```bash
pnpm --filter @refrain/extract add pdfjs-dist tesseract.js
```

```ts
// packages/extract/src/pdf.ts
import type { TextPage } from "./types"

/** Confidence for a page we read a real text layer from. */
export const TEXT_LAYER_CONFIDENCE = 0.95
/** Below a page threshold, there is no text layer. Route to OCR. */
export const SCAN_THRESHOLD_CHARS = 80

let pdfjs: typeof import("pdfjs-dist") | null = null

export async function initPdfjs() {
  if (pdfjs) return pdfjs
  pdfjs = await import("pdfjs-dist")
  // MUST be a bundled URL, never a CDN path. MV3 forbids remote code.
  pdfjs.GlobalWorkerOptions.workerSrc = new URL(
    "pdfjs-dist/build/pdf.worker.mjs",
    import.meta.url,
  ).toString()
  return pdfjs
}

export async function readPdf(bytes: ArrayBuffer): Promise<{
  pages: TextPage[]
  width: number
  height: number
}> {
  const lib = await initPdfjs()
  // Copy. pdf.js TAKES OWNERSHIP of this buffer and detaches it.
  // Pass the original and your caller's copy becomes unusable —
  // a genuinely baffling bug if you then try to store it.
  const doc = await lib.getDocument({ data: new Uint8Array(bytes.slice(0)) }).promise

  const pages: TextPage[] = []

  for (let n = 1; n <= doc.numPages; n++) {
    const page = await doc.getPage(n)
    const text = await reconstructLines(page)
    const viewport = page.getViewport({ scale: 1 })

    pages.push({
      pageNumber: n,
      text,
      // ← The routing signal. One number decides the engine.
      confidence: text.length < SCAN_THRESHOLD_CHARS ? 0 : TEXT_LAYER_CONFIDENCE,
      width: viewport.width,
      height: viewport.height,
    })
  }

  return { pages, width: doc.numPages, height: 0 }
}
```

> **`bytes.slice(0)` is not paranoia.** `pdf.js` transfers the `ArrayBuffer` to its worker and
> **detaches it.** Your `ArrayBuffer.byteLength` becomes `0` afterwards. If you passed the same
> buffer you were about to encrypt and store, you have silently lost the document. This bug
> presents as "the upload saved as an empty file."

### Reconstructing lines — the whole job

`pdf.js` does not give you lines. It gives you **positioned text fragments in whatever order
the PDF writer emitted them.** Without reconstructing lines you get
`"Marksheet Rohit Jadhav 87.4"` on one line and no regex on earth can pull the name out.

```ts
async function reconstructLines(page: any): Promise<string> {
  const content = await page.getTextContent()

  /** y → fragments on that line. Rounded because glyphs jitter. */
  const lines = new Map<number, string[]>()

  for (const item of content.items) {
    if (!("str" in item) || !item.str) continue
    // transform[5] is the y position in PDF user space.
    // PDF y grows UPWARD, so sort descending to get reading order.
    const y = Math.round((item.transform[5] as number) / 3)
    if (!lines.has(y)) lines.set(y, [])
    lines.get(y)!.push(item.str)
  }

  return [...lines.entries()]
    .sort((a, b) => b[0] - a[0])                      // top of page first
    .map(([, fragments]) => {
      // Sort fragments left-to-right WITHIN the line by their x.
      return fragments.join(" ").replace(/\s+/g, " ").trim()
    })
    .filter(Boolean)
    .join("\n")
}
```

**Four details, each of which costs you a day if you miss it:**

**`transform[5]` is y, in PDF user space.** Dividing by 3 approximates a line height and
buckets glyphs into lines. Different PDFs use different sizes, so tune the divisor against real
documents.

**Sort descending.** PDF's origin is bottom-left, so y increases upward. Sorting ascending gives
you the page **upside down**. You will notice immediately.

**Sort fragments within a line by x, not by array order.** A table row is many `transform`
entries in arbitrary emission order. Without x-sorting, `87.4` can land before the name.

**`join("\n")` between lines and `" "` within them.** This is what makes your regexes work —
a pattern anchored to `^` and `$` per line can only work if lines exist.

```ts
// Verify against a real document, immediately.
const { pages } = await readPdf(bytes)
console.log(pages[0]!.text.split("\n").slice(0, 12).join("\n"))
```

> **Read the output before writing any regex.** If the lines are scrambled, every extractor
> rule you write next will fail, and you will conclude the regexes are wrong. They are not. Look
> at the text first. It takes thirty seconds and saves an hour.

---

## Step 3 — The scan path

```ts
// packages/extract/src/rasterise.ts

/** pdf.js page → canvas. Tesseract needs pixels, not PDF bytes. */
export async function pageToCanvas(
  doc: any, pageNumber: number, scale = 2,
): Promise<HTMLCanvasElement> {
  const page = await doc.getPage(pageNumber)
  // getViewport accounts for the PDF's own rotation. Do not add
  // your own transform on top — double-rotated pages OCR as garbage.
  const viewport = page.getViewport({ scale })

  const canvas = document.createElement("canvas")
  canvas.width = Math.floor(viewport.width)
  canvas.height = Math.floor(viewport.height)

  const ctx = canvas.getContext("2d")!
  // Tesseract needs dark-on-light. An inverted page reads as noise.
  ctx.fillStyle = "#ffffff"
  ctx.fillRect(0, 0, canvas.width, canvas.height)

  await page.render({ canvasContext: ctx, viewport }).promise
  return canvas
}

/**
 * A standalone photo (user dropped a JPG of a marksheet).
 * `imageOrientation: "from-image"` applies the EXIF rotation.
 *
 * Without this, a portrait phone photo of a landscape document
 * arrives sideways, OCR reads sideways, and produces a confident
 * wrong answer.
 */
export async function imageToCanvas(file: Blob, scale = 2): Promise<HTMLCanvasElement> {
  const bitmap = await createImageBitmap(file, { imageOrientation: "from-image" })
  const canvas = document.createElement("canvas")
  canvas.width = Math.floor(bitmap.width * scale)
  canvas.height = Math.floor(bitmap.height * scale)

  const ctx = canvas.getContext("2d")!
  ctx.fillStyle = "#ffffff"
  ctx.fillRect(0, 0, canvas.width, canvas.height)
  ctx.drawImage(bitmap, 0, 0, canvas.width, canvas.height)
  bitmap.close()
  return canvas
}
```

> **`scale = 2` is the minimum and it is not arbitrary.** Under 2×, OCR on a phone photo is
> unusable. Over 3× it is slow enough to hang the panel on a mid-range laptop. Note also the
> white background fill — **Tesseract expects dark text on light.** A transparent or coloured
> background reads as noise and confidence collapses without any error.

---

## Step 4 — Tesseract, without leaking

```ts
// packages/extract/src/ocr.ts
import type { TextPage } from "./types"

/** OCR is never trustworthy enough to auto-fill. Below the threshold. */
export const OCR_CONFIDENCE = 0.62

/** Yield to the main thread so the panel stays responsive. */
const breathe = () => new Promise<void>((r) => setTimeout(r, 0))

export interface OcrProgress {
  page: number
  total: number
}

export async function ocrDocument(
  canvases: Array<{ pageNumber: number; canvas: HTMLCanvasElement }>,
  onProgress?: (p: OcrProgress) => void,
): Promise<TextPage[]> {
  const { createWorker } = await import("tesseract.js")

  // ONE worker for the whole document. Not one per page.
  const worker = await createWorker("eng")
  const out: TextPage[] = []

  try {
    for (const [i, { pageNumber, canvas }] of canvases.entries()) {
      onProgress?.({ page: pageNumber, total: canvases.length })

      const { data } = await worker.recognize(canvas)
      out.push({ pageNumber, text: data.text, confidence: OCR_CONFIDENCE, width: canvas.width, height: canvas.height })

      await breathe()   // ← the panel is frozen without this
    }
    return out
  } finally {
    await worker.terminate()   // ← the bug that kills tabs
  }
}
```

### Three things that will ruin you if you skip them

**`worker.terminate()` in a `finally`.** A six-page scanned marksheet is six live workers if
you leak them. Tesseract workers each hold a WASM heap of tens of megabytes. The browser tab
climbs past 2GB and **the renderer process is killed with no stack trace and no console error.**
This is the single most common Tesseract bug and it presents as "the extension crashed."

**One worker for the whole document.** `createWorker` compiles and loads the language model.
Calling it per page reloads it six times — several seconds of pure waste each time.

**`await breathe()` between pages.** OCR is synchronous CPU work in a WASM thread. Without
yielding, the panel is frozen for the entire extraction, the mascot animation stops, and the
user closes the tab. Yielding every page keeps the UI at ~60fps and lets the progress bar move.

```ts
// One progress line of code, and it is the difference between
// "broken" and "slow":
export async function extractWithProgress(file: File, onProgress: (p: string) => void) {
  const bytes = await file.arrayBuffer()
  const doc = await openDoc(bytes)

  const textPages = []
  const toOcr: number[] = []

  for (const p of doc.pages) {
    if (p.confidence === 0) toOcr.push(p.pageNumber)
    else textPages.push(p)
  }

  if (toOcr.length === 0) return textPages

  onProgress(`Reading ${toOcr.length} scanned page${toOcr.length > 1 ? "s" : ""}…`)

  const canvases = []
  for (const n of toOcr) canvases.push({ pageNumber: n, canvas: await pageToCanvas(doc.raw, n, 2) })

  return [...textPages, ...await ocrDocument(canvases, ({ page, total }) =>
    onProgress(`Page ${page} of ${total}`))]
}
```

---

## Step 5 — Fact extraction

```ts
// packages/extract/src/rules.ts
import type { Provenance } from "@refrain/fields"

export interface ExtractedFact {
  key: string
  value: string
  confidence: number
  provenance: Provenance
}

/**
 * First match wins, page order ascends. Never overwrite — a later
 * page is not more authoritative than an earlier one.
 */
const RULES: Array<{ key: string; re: RegExp }> = [
  { key: "fullName",          re: /(?:student'?s?\s+)?name\s*[:\-–]?\s*([A-Z][a-zA-Z]+(?:\s+[A-Z][a-zA-Z]+){1,3})/ },
  { key: "dateOfBirth",       re: /(?:d\.?o\.?b\.?|date\s+of\s+birth|date\s+of\s+birth)\s*[:\-–]?\s*(\d{1,2}[/\-.]\d{1,2}[/\-.]\d{4})/i },
  { key: "tenthPercentage",   re: /(?:class\s*x\b|10th|matric|secondary)[^\d%\n]{0,24}(\d{2}\.?\d?)\s*%/i },
  { key: "twelfthPercentage", re: /(?:class\s*xii\b|12th|intermediate|higher\s+secondary)[^\d%\n]{0,24}(\d{2}\.?\d?)\s*%/i },
  { key: "rollNumber",        re: /(?:roll|reg|enroll|seat)\s*(?:no\.?|number)?\s*[:\-–]?\s*([A-Z0-9]{6,})/i },
  { key: "university",        re: /(?:university|college|institution|school)\s*[:\-–]?\s*([^\n]{4,60})/i },
  { key: "cgpa",              re: /(?:cgpa|gpa|aggregate)[^\d\n]{0,12}(\d\.\d{1,2})/i },
  { key: "pan",               re: /\b([A-Z]{5}\d{4}[A-Z])\b/ },
  { key: "aadhaar",           re: /\b([2-9]\d{3}\s?\d{4}\s?\d{4})\b/ },
]

/**
 * `locator` carries the page AND the matched line, so the user can
 * verify in two seconds. This is not decoration — it is what makes
 * an untrusted extraction acceptable at all.
 */
export function extractFacts(
  pages: Array<{ pageNumber: number; text: string; confidence: number }>,
  filename: string,
): ExtractedFact[] {
  const out: ExtractedFact[] = []
  const seen = new Set<string>()

  for (const page of pages) {
    for (const line of page.text.split("\n")) {
      for (const { key, re } of RULES) {
        if (seen.has(key)) continue
        const m = line.match(re)
        const value = m?.[1]?.trim()
        if (!value) continue

        seen.add(key)
        out.push({
          key,
          value,
          confidence: page.confidence,
          provenance: {
            kind: "document",
            label: filename,
            locator: `p.${page.pageNumber} · "${truncate(line, 48)}"`,
            extractedAt: new Date().toISOString(),
          },
        })
      }
    }
  }
  return out
}

const truncate = (s: string, n: number) =>
  s.length <= n ? s : s.slice(0, n - 1).replace(/\s+\S*$/, "") + "…"
```

**Why the locator includes the matched line, not just the page number.** Page 3 of a
six-page document is still a hunt. `"p.3 · "Class X Aggregate: 87.4%""` is a glance. The
difference is three seconds versus thirty, and thirty seconds is the difference between a user
confirming and a user giving up.

**Why `aadhaar` and `pan` are in the extractor at all.** You are *reading* them from a document
the user already has. §11's no-list is about *submitting* identity data, not about a user
reading their own Aadhaar number off their own PDF on their own machine. Do not include them in
the auto-filled field set — see the filter below.

```ts
/** Fields that may ever reach a form. Identity numbers are stored
 *  for the user's reference but never auto-filled. */
export const FILLABLE = new Set([
  "fullName", "dateOfBirth", "tenthPercentage", "twelfthPercentage",
  "cgpa", "university", "rollNumber",
])

export const fillableFacts = (facts: ExtractedFact[]) =>
  facts.filter((f) => FILLABLE.has(f.key))
```

---

## Step 6 — The confirm gate, and provenance promotion

This is the component that makes OCR safe.

```tsx
// apps/web/src/routes/Documents.tsx — the review
import { useState } from "react"
import { ProvenanceChip } from "@refrain/ui/provenance"
import { REVIEW_THRESHOLD } from "@refrain/fields"

export function ExtractionReview({
  facts, filename, onCommit,
}: {
  facts: ExtractedFact[]
  filename: string
  onCommit: (facts: ExtractedFact[]) => Promise<void>
}) {
  const [confirmed, setConfirmed] = useState<Set<string>>(new Set())
  const [drafts, setDrafts] = useState<Record<string, string>>(
    Object.fromEntries(facts.map((f) => [f.key, f.value])),
  )

  const low = facts.filter((f) => f.confidence < REVIEW_THRESHOLD)
  const allConfirmed = confirmed.size >= facts.length

  return (
    <div className="rounded-panel bg-surface-muted p-5">
      <h2 className="font-semibold text-ink">
        {facts.length} field{facts.length === 1 ? "" : "s"} found in {filename}
      </h2>

      {low.length > 0 ? (
        <p className="mt-1 text-sm text-ink-muted">
          This is a scan, so the reading is not certain. Check each one — after that it is
          exact forever, and Refrain will never ask again.
        </p>
      ) : (
        <p className="mt-1 text-sm text-ink-muted">
          Check these before they go into any form.
        </p>
      )}

      <ul className="mt-4 flex flex-col gap-2">
        {facts.map((f) => {
          const dirty = drafts[f.key] !== f.value
          const isConfirmed = confirmed.has(f.key)
          return (
            <li key={f.key} className="flex items-start gap-3 rounded-card bg-surface p-3">
              <div className="min-w-0 flex-1">
                <span className="text-xs text-ink-muted">{humanKey(f.key)}</span>
                <input
                  value={drafts[f.key]}
                  onChange={(e) => setDrafts((d) => ({ ...d, [f.key]: e.target.value }))}
                  onBlur={() => setConfirmed((c) => new Set(c).add(f.key))}
                  className="w-full rounded border border-surface-muted bg-transparent px-2 py-1 text-ink"
                />
                {/* Click the chip to open the source. That is the whole
                    verification loop and it must be one click. */}
                <button onClick={() => openSource(f.provenance.locator)}
                  className="mt-1 block">
                  <ProvenanceChip
                    provenance={f.provenance}
                    locked={!isConfirmed && !dirty}
                    editedByUser={isConfirmed}
                  />
                </button>
              </div>

              {(!isConfirmed && !dirty) && (
                <button onClick={() => setConfirmed((c) => new Set(c).add(f.key))}
                  className="shrink-0 rounded-card border px-3 py-1.5 text-xs font-medium text-ink">
                  Looks right
                </button>
              )}
            </li>
          )
        })}
      </ul>

      <button
        disabled={!allConfirmed}
        onClick={() => onCommit(facts.map((f) => ({
          ...f,
          value: drafts[f.key],
          // ← PROMOTION. See below.
          confidence: 1,
          provenance: { ...f.provenance, kind: "manual" as const, label: `${filename} (you checked)` },
        })))}
        className="mt-4 rounded-card bg-brand-500 px-4 py-2 text-sm font-medium text-white disabled:opacity-50"
      >
        Save to my profile
      </button>
    </div>
  )
}
```

### The promotion is the entire design

```ts
// Before
{ value: "Rohit Jadhav", confidence: 0.62, provenance: { kind: "document", locator: "p.3" } }

// After one tap
{ value: "Rohit Jadhav", confidence: 1.0,  provenance: { kind: "manual", label: "marksheet.pdf (you checked)" } }
```

Three things change, and all three matter:

**Confidence → 1.0.** It is no longer an extraction. It is a user-asserted fact. It fills
everywhere, immediately, forever.

**`kind` → `manual`, not `document`.** The provenance must reflect *what the fact is now*, not
where the bytes came from. A fact that says "from marksheet.pdf, confidence 1.0" is claiming
the document was 100% correct. It was not — **you** were.

**The label keeps the document name, plus "you checked."** You did not lose the audit trail.
You gained an audit trail that now includes the human.

> **This is why `SourceKind` includes `manual` in Chapter 2's Zod enum.** It is not a
> lower-quality `document`. It is a *different kind of truth*, and it deserves its own chip and
> its own colour. Chapter 9's `ProvenanceChip` already renders it as "you typed this" — which
> is honest, because from the system's perspective, it is.

---

## Step 7 — Storage

```ts
// packages/vault/src/documents.ts
import { db } from "./db/schema"
import { sealBytes, unsealBytes } from "./crypto/aead"
import { extractFacts, fillableFacts } from "@refrain/extract"
import type { Provenance } from "@refrain/fields"

export async function storeExtracted(
  file: File,
  kind: DocumentRecord["kind"],
  key: CryptoKey,
  facts: ExtractedFact[],
): Promise<DocumentRecord> {
  const record: DocumentRecord = {
    id: crypto.randomUUID(),
    kind,
    filename: file.name,
    mimeType: file.type,
    byteLength: file.size,
    addedAt: Date.now(),
  }

  // Bytes, not a stringified object. See the top of the chapter.
  const sealed = await sealBytes(key, await file.arrayBuffer())
  await db.documents.add(record)
  await db.blobs.add({ id: record.id, recordId: record.id, sealed })

  for (const f of facts) {
    await db.facts.put({
      key: f.key,
      value: f.value,
      confidence: f.confidence,
      provenance: f.provenance,
      updatedAt: Date.now(),
    })
  }
  return record
}

export async function readDocumentBytes(id: string, key: CryptoKey) {
  const row = await db.blobs.get(id)
  if (!row) throw new Error("document not found")
  return unsealBytes(key, row.sealed)
}
```

**The metadata row holds no PII beyond the filename.** That is what lets you index it, list it,
and sort it without decrypting anything. `filename` is arguably PII
(`rohit_jadhav_aadhaar.pdf`) — accept it, do not pretend it is not there, and document it.

---

## Step 8 — Test against real documents

This is the chapter where "works on my machine" is most dangerous. **Get real documents.**

```bash
# A folder of real marksheets, transcripts, and statements — yours and
# friends', redacted where needed. Twenty files beats two.
mkdir -p ~/refrain-samples
```

| Test | What it catches |
|---|---|
| A modern digital PDF | Line reconstruction, worker setup |
| A 2015 government PDF with broken font encoding | Mojibake, missing `str` fields |
| A **scanned** marksheet | Scan detection, Tesseract, the 0.62 rule |
| A **sideways photo** | EXIF orientation — produces a *confident wrong answer* |
| A **two-column** layout | x-sorting within lines |
| A PDF with a `maxLength` table | Whitespace and line density |
| A **folded, creased, low-light** photo | The real worst case. Confidence should be visibly lower |

```ts
// packages/extract/src/extract.test.ts — run against the folder
describe("against real documents", () => {
  it.each(SAMPLES)("%s produces readable lines", async (name) => {
    const { pages } = await readPdf(await readFile(name))
    for (const p of pages) {
      // Every non-scan page must reconstruct into real lines.
      if (p.confidence === 0) continue
      expect(p.text).not.toMatch(/(.{200})\1/)       // no duplicated blocks
      expect(p.text.length).toBeGreaterThan(SCAN_THRESHOLD_CHARS)
    }
  })
})
```

> **The duplicated-block assertion catches the classic pdf.js failure.** When font encoding
> breaks, `getTextContent` returns the same fragment over and over and you get a page of
> `"fffff fffff fffff"` that is confidently long enough to skip the OCR branch. Your extractor
> then regexes against garbage and finds nothing — and reports zero fields, which looks like a
> vault problem, not a parser problem.

---

## Step 9 — Commit

```bash
git add -A
git commit -m "feat(extract): pdf text layer + line reconstruction, scan routing, OCR at 0.62, confirm gate"
```

Add two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | OCR confidence fixed at 0.62, never tuned upward | One confirmation per fact converts untrusted automation into trusted data. The safety is the number, not the accuracy. |
| 2026-10-XX | Confirmed facts promote to `manual` provenance at 1.0 | The provenance must describe what the fact *is now* — user-asserted — not where the bytes came from. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`reconstructLines` and the four details** | **You.** This is the core competence |
| **`SCAN_THRESHOLD_CHARS = 80` and why one number routes everything** | **You** |
| **`OCR_CONFIDENCE = 0.62` and the promotion logic** | **You.** Safety decision |
| `sealBytes` / `unsealBytes` | **You** — never paste a byte path |
| `pageToCanvas` scale and white-background fill | **You**, then tune against a real scan |
| `imageToCanvas` and the EXIF flag | **You** |
| The `RULES` regexes | **You**, testing against your own real documents |
| `filerableFacts` and the identity-number exclusion | **You** |
| `worker.terminate()` and the `breathe()` yield | **OpenCode**, then add your own leak test |
| `extractWithProgress` and the orchestrator | **OpenCode** |
| `storeExtracted` / `readDocumentBytes` | **OpenCode** |
| `ExtractionReview` markup | **OpenCode** — you write the copy and the promotion |

---

## Gotchas in this chapter

**`No GlobalWorkerOptions.workerSrc specified`.** You skipped `initPdfjs`, or the path does not
resolve as a bundled URL. A CDN path is not just wrong — MV3 rejects remote code.

**The caller's `ArrayBuffer` becomes zero-length.** `pdf.js` transfers and detaches it. Pass
`bytes.slice(0)`.

**Every line of extracted text is reversed.** You sorted y ascending. PDF y grows upward.

**Words are in the wrong order within a line.** You did not sort fragments by `x` inside each
line bucket.

**One page of `"fffff fffff fffff"`.** Broken font encoding. Check `getTextContent()` output
raw, before any extraction. Nothing downstream will work until this does.

**The tab crashes at 2GB after a scanned PDF.** Leaked Tesseract workers. `finally { terminate() }`.

**The panel is frozen for 30 seconds.** No `await breathe()`. Also check you are OCR-ing six
pages when one would do — crop to the relevant region if you can.

**A sideways photo produces a confident wrong name.** Missing
`createImageBitmap(blob, { imageOrientation: "from-image" })`. This is the worst bug in the
chapter because the value is *confident* and *wrong*.

**The stored document is 0 bytes.** You passed the same `ArrayBuffer` to `pdf.js` and then to
`sealBytes`. `slice(0)` it.

**No facts extracted from a readable PDF.** The lines exist but your regexes assume a label
followed by a colon. Real documents use tabs, dots, and varying spacing. Print the lines and
read them.

**Your 20-document test set is three documents.** Two are the easy case. The scanned, creased,
sideways photo is the one that finds the bugs — find or make it before you ship.

---

## Verify before moving on

- [ ] `grep -rE "https?://" apps/extension/.output/chrome-mv3/manifest.json` — no remote code
- [ ] A digital PDF produces correct, correctly-ordered lines
- [ ] A scanned PDF routes to OCR automatically
- [ ] A sideways photo is auto-rotated and OCRs correctly
- [ ] OCR confidence is 0.62 and **no OCR value ever auto-fills a form**
- [ ] Every fact has `locator` with page number and the matched line
- [ ] Clicking the provenance chip opens the source at that page
- [ ] Confirming a fact promotes it to `manual` at 1.0
- [ ] A promoted fact fills automatically on a form with **no** warning chip
- [ ] The panel stays responsive during extraction
- [ ] Tesseract workers terminate — check the tab's memory after 5 extractions
- [ ] The stored blob round-trips: read it back and open it in a PDF viewer
- [ ] Identity numbers are stored but excluded from `FILLABLE`

---

## Check yourself before Chapter 18

> **This is the last frontend chapter.** Chapter 4 opens the backend, and with it the
> uncomfortable work of rewriting §11 to tell the truth about a server existing. If any of these
> ten answers is shaky, this is the last cheap moment to fix it.

1. **Why is the OCR confidence 0.62 and not, say, 0.85?**
2. **What does the promotion to `manual` at 1.0 actually buy the user?**
3. **Why does `locator` include the matched line, not just the page number?**
4. **Why must you `slice(0)` the ArrayBuffer before passing it to `pdf.js`?**
5. **What does the white background fill do to Tesseract accuracy?**
6. **What happens to memory without `worker.terminate()`, and how does it present?**
7. **Why is `aadhaar` in the extractor but not in `FILLABLE`?**
8. **Why is a `manual` fact labelled differently from the document it came from?**

---

**Next: [Chapter 18 — Backend Foundation](./04-backend-foundation.md)** — what MongoDB changes,
what §11 now has to say, and how to build a sync backend without giving up end-to-end encryption.