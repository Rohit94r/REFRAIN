# Chapter 14 — The Mapping Engine

> **Day 15 · Goal: all 14 fields mapped, rule-based, with a confidence score and a reason.**
>
> This chapter contains the moat. Everything else in the product is plumbing. This is the part
> nobody else has, and the part that makes a fill correct instead of plausible.

---

## Understand this first

### Rules first, models second, and never the reverse

The instinct is to reach for an LLM immediately. Do not. There are three kinds of mapping job
in this product and only one of them needs a model:

| Job | Example | Best tool | Cost | Explainable? |
|---|---|---|---|---|
| **Exact & alias** | "Email address" → `email` | Lookup table | ₹0 | Yes, trivially |
| **Variant normalisation** | "10th Percentage" → `tenth_percentage` | Normalise + rules | ₹0 | Yes |
| **Genuinely ambiguous** | "Aggregate % (Class X)" → CGPA? %? | Local model | ₹0 if local | Loosely |

**Rule-based first is not a compromise. It is strictly better for two thirds of the work:**

- **₹0 per fill.** This is the §11 local-first arithmetic. Every cloud call spends the moat.
- **Instant.** A rules pass on 40 fields is sub-millisecond. A model call is 400ms–2s.
- **Explainable.** "Mapped because label matched alias `emailaddr`" is a sentence you can put
  in a chip. "Mapped because the model thought so" is not, and §11 requires provenance you
  can *show*.

**The confidence score is not a probability. It is a statement about which layer produced the
answer.** That is why the `Confidence.reason` field in `@refrain/fields` is a required string:
you must always be able to say *why* you are this confident.

### "Do not guess" is an engineering rule, not an attitude

From §11: no auto-submit, no CAPTCHA bypass, no bulk submission. The one people forget is the
statistical version:

> **A 70%-confident name filled into a government scholarship form is a wrong answer with a
> confident interface.**

So the resolver has a hard gate. Below `REVIEW_THRESHOLD`, the output is not a value at all —
it is `needs-input`, and the review screen asks the user. `isSafeToFill` from Chapter 12 is
that gate, and it lives in the vault so there is exactly one place to audit it.

**This is the single decision that separates Refrain from every "AI autofill" product.** They
optimise for "how many fields did we fill?" You optimise for "how many fields did we fill
correctly that the user did not have to correct?" Those are different metrics and yours is
smaller. Say so in the README.

### The hand-labelled dataset is the moat

Not the code. Not the model. **A labelled JSON file.**

```jsonc
// packages/mapping/data/aliases.json
{
  "email": ["email", "email address", "e-mail", "email id", "mail id",
            "your email", "emailaddress", "primary email"],
  "phone": ["phone", "mobile", "mobile number", "contact number", "phone no",
            "contact", "your mobile number", "cell number", "whatsapp number"],
  "tenth_percentage": ["10th percentage", "10th %", "class 10 percentage",
                       "class 10 marks", "10th board percentage", "matric percentage"],
  "cgpa": ["cgpa", "current cgpa", "gpa", "aggregate", "cgpa out of 10",
           "graduation cgpa", "cgpa (out of 10)"]
}
```

Three hundred of these pairs, hand-written, each one an afternoon of looking at real forms.
**That is the asset.** A competitor can clone your repo in a day. They cannot clone the four
months you spent noticing that Indian portals say `Class X Aggregate` for what everyone else
calls GPA.

This is why the Playwright script in Chapter 13 was the highest-leverage 40% in that chapter.
It generates the raw material. The labelling is yours, and it is slow, and it is the product.

### Why this must be a pure function

```ts
function resolve(schema: FormSchema, profile: Profile): MappingResult[]
```

No DOM. No IndexedDB. No network. Takes data, returns data.

This is not architectural purity for its own sake. It means your entire moat is testable in
40ms with `vitest`, which means you can **replay 40 saved real forms against every change you
make and see whether the score went up or down.** Without that, every mapping improvement is a
guess, and you will ship regressions while fixing bugs.

---

## Step 1 — The normalisation layer

Everything below this layer is string equality. This layer is what makes that possible.

```ts
// packages/mapping/src/normalize.ts

/** Common noise in Indian form labels that carries no signal. */
const NOISE = [
  /\(required\)/gi, /\(optional\)/gi, /\(mandatory\)/gi, /\*+/g,
  /^\s*(mr|mrs|ms|dr|prof)\.?\s+/i,
  /\b(name|enter|provide|please\s+enter|type|fill)\b/gi,
  /\b(of\s+the\s+)?(applicant|candidate|student|you)\b/gi,
  /[?!]/g,
]

/**
 * One canonical string per label. Two labels normalise to the same
 * value if and only if they mean the same question.
 */
export function normalizeLabel(raw: string): string {
  let s = raw.toLowerCase().trim()
  s = s.normalize("NFKD").replace(/[\u0300-\u036f]/g, "")  // diacritics: "Marksheets" → "marksheets"
  for (const re of NOISE) s = s.replace(re, " ")
  return s.replace(/[^a-z0-9%\s./-]/g, " ").replace(/\s+/g, " ").trim()
}
```

| Raw label | Normalised |
|---|---|
| `* Full Name of the Applicant (Required)` | `full name` |
| `Email address` | `email address` |
| `Mobile No.` | `mobile no` |
| `10th % (Class X)` | `10th % class x` |
| `Enter your CGPA out of 10` | `cgpa out of 10` |
| `ईमेल पता` | *(empty — see below)* |

**Devanagari and regional-language labels need an explicit answer, not a hope.** Many Indian
government portals have bilingual labels. Your options:

1. **Drop them.** Then you report a field with an unreadable label and the user fills it by hand.
   Honest, and it works today.
2. **Translate with a local model.** Feasible but adds a whole failure mode.
3. **A small hand-written Hindi/regional synonym set.** Maybe 60 entries. Highest value per
   hour of any task in this chapter, because these are exactly the forms nobody else handles.

**Do option 1 now, and treat option 3 as labelling work that never finishes** — it is the
same `aliases.json` this chapter builds. Record the deferral. Silently producing garbage for
Devanagari labels is the failure mode to avoid — `normalizeLabel` returning `""` means the
field falls through to `needs-input`, which is correct.

### Synonyms: the two-way index

```ts
export function buildAliasIndex(aliases: Record<string, string[]>) {
  const exact = new Map<string, string>()
  const canonical = new Map<string, string>()   // normalised canonical key → itself

  for (const [key, variants] of Object.entries(aliases)) {
    const normKey = normalizeLabel(key)
    canonical.set(normKey, key)
    exact.set(normKey, key)

    for (const v of variants) {
      const n = normalizeLabel(v)
      if (n) exact.set(n, key)
    }
  }
  return { exact, canonical }
}
```

> **Normalise the canonical keys too, not just the aliases.** Otherwise `tenth_percentage`
> and `10th_percentage` end up as two different keys in your profile and the user fills in
> both. This is exactly the "drift bug" §10 warns about, and the fix is one line here.

---

## Step 2 — The rule layers

```ts
// packages/mapping/src/rules.ts
import { REVIEW_THRESHOLD } from "@refrain/fields"
import type { Confidence, FormSchema, MappingResult, ProfileFact } from "@refrain/fields"

interface Candidate {
  key: string
  fact: ProfileFact
  confidence: Confidence
}

/** Layer 1 — exact normalised match. Fastest, most trustworthy. */
function layerExact(label: string, idx: ReturnType<typeof buildAliasIndex>): Candidate | null {
  const key = idx.exact.get(label)
  if (!key) return null
  return { key, fact: null!, confidence: { score: 1.0, reason: `Exact label match` } }
}

/**
 * Layer 2 — containment + alias scan. "E-mail address of applicant" → email.
 *
 * Scored by how much of the alias matched, so a long label that contains a
 * short alias scores lower than a tight match. Contains is a weaker signal
 * than equality and the number has to say so.
 */
function layerContainment(
  label: string,
  profile: Map<string, ProfileFact>,
  aliases: Record<string, string[]>,
): Candidate[] {
  const out: Candidate[] = []

  for (const [key, variants] of Object.entries(aliases)) {
    const fact = profile.get(key)
    if (!fact) continue

    const needles = [normalizeLabel(key), ...variants.map(normalizeLabel)].filter(Boolean)

    for (const needle of needles) {
      if (!label.includes(needle)) continue

      // Coverage: how much of the label the alias accounts for.
      const coverage = needle.length / Math.max(label.length, 1)
      const score = 0.72 + 0.2 * Math.min(coverage * 2, 1)

      out.push({
        key,
        fact,
        confidence: {
          score: Math.min(score, 0.95),
          reason: `Label contains "${needle}"`,
        },
      })
      break
    }
  }
  return out.sort((a, b) => b.confidence.score - a.confidence.score)
}

/**
 * Layer 3 — the profile tells us what it has. Match the label against
 * attribute KEYS the user actually filled, not just the canonical list.
 *
 * This is how "Class X Aggregate" reaches a fact the user typed under
 * an attribute key nobody anticipated.
 */
function layerAttributeKeys(label: string, profile: Profile): Candidate[] {
  const out: Candidate[] = []
  for (const [attrKey, fact] of Object.entries(profile.attributes)) {
    const n = normalizeLabel(attrKey.replace(/_/g, " "))
    if (!n || !label.includes(n)) continue
    out.push({
      key: attrKey,
      fact,
      confidence: {
        score: 0.78,
        reason: `Matched your saved field "${attrKey.replace(/_/g, " ")}"`,
      },
    })
  }
  return out
}
```

**Why layer 3 exists and why it scores lower.** It reaches facts no rule anticipated, which
is genuinely useful — but it is matching on a key the *user* named, not on a field label we
validated. 0.78 is above `REVIEW_THRESHOLD` (0.75) so it fills, and it lands in the review
screen with a chip the user can see and correct. If you scored it 0.95 it would look
authoritative. **The score is a communication to the user, not a probability.**

---

## Step 3 — Type coercion

The other half of correctness. `87.4` is a CGPA or a percentage or a total marks — the label
decides, and the two are not interchangeable.

```ts
// packages/mapping/src/coerce.ts

/** Percentages arrive as "87.4", "87.4%", "87.4 %", "87.40", "—". */
export function toNumber(value: string): number | null {
  const cleaned = value.replace(/[^\d.-]/g, "")
  if (!cleaned || cleaned === "-" || cleaned === ".") return null
  const n = Number(cleaned)
  return Number.isFinite(n) ? n : null
}

/**
 * CGPA is almost always out of 10 or 4 in Indian forms. A user who
 * stores 8.7 will be asked for "out of 4" and needs 3.48, not 8.7.
 * Silently writing 8.7 into an "out of 4" field is worse than asking.
 */
export function toCGPA(value: string, outOf: number): string | null {
  const n = toNumber(value)
  if (n === null) return null
  const from = value.includes("%") || n > 10 ? 10 : 10 // source scale
  const converted = (n / from) * outOf
  return String(Number(converted.toFixed(2)))
}

/** Dates: Indian forms use at least six layouts. */
export function toISODate(value: string): string | null {
  const v = value.trim()

  const named: Record<string, string> = {
    today: new Date().toISOString().slice(0, 10),
  }
  if (named[v.toLowerCase()]) return named[v.toLowerCase()]!

  // dd/mm/yyyy, dd-mm-yyyy, dd.mm.yyyy
  let m = v.match(/^(\d{1,2})[/\-.](\d{1,2})[/\-.](\d{4})$/)
  if (m) {
    const [, d, mo, y] = m
    if (+d > 12) return `${y}-${pad(mo)}-${pad(d)}`       // unambiguous
    return `${y}-${pad(mo)}-${pad(d)}`                      // assume DD/MM — see note
  }

  // "14 July 2006"
  m = v.match(/^(\d{1,2})\s+([a-z]+)\s+(\d{4})$/i)
  if (m) {
    const month = MONTHS[m[2]!.toLowerCase()]
    if (month) return `${m[3]}-${pad(month)}-${pad(m[1])}`
  }

  // ISO already
  m = v.match(/^(\d{4})-(\d{2})-(\d{2})$/)
  if (m) return v

  return null   // ← null means needs-input. NEVER a guess.
}

const MONTHS: Record<string, string> = {
  january: "01", february: "02", march: "03", april: "04",
  may: "05", june: "06", july: "07", august: "08",
  september: "09", october: "10", november: "11", december: "12",
}
const pad = (s: string) => s.padStart(2, "0")
```

> **`DD/MM` vs `MM/DD` is genuinely ambiguous and you must pick one.** Pick `DD/MM`, because
> your users are Indian, and because `14/07` is unambiguous anyway (14 > 12) while `07/08`
> is not. Put the assumption in `refrain.md` §19 and surface a chip for dates you inferred.
> Silently choosing wrong is the only unacceptable outcome.

**Always return `null`, never a best guess.** `null` becomes `needs-input`, the user types
three characters, and nobody is harmed. A wrong number on a scholarship form is a rejection
letter.

---

## Step 4 — The orchestrator

```ts
// packages/mapping/src/resolve.ts

export function resolveField(
  field: FormFieldSchema,
  profile: Profile,
  facts: Map<string, ProfileFact>,
  idx: ReturnType<typeof buildAliasIndex>,
  aliases: Record<string, string[]>,
): MappingResult {
  const label = normalizeLabel(field.label || field.name || "")

  // Layer 1
  const exact = layerExact(label, idx)
  if (exact) return finish(field, facts.get(exact.key), exact.confidence)

  // Layer 2
  const contained = layerContainment(label, facts, aliases)
  if (contained.length) {
    const best = contained[0]!
    // Two aliases matched different keys almost equally — genuinely ambiguous.
    const runnerUp = contained[1]
    if (runnerUp && Math.abs(best.confidence.score - runnerUp.confidence.score) < 0.05) {
      return {
        schema: field,
        value: null,
        status: "needs-review",
        // The reason string is what the chip shows. Make it actionable.
        // ...and name the runner-up so the user is not left guessing.
      }
    }
    return finish(field, best.fact, best.confidence)
  }

  // Layer 3
  const attrs = layerAttributeKeys(label, profile)
  if (attrs.length) {
    return finish(field, attrs[0]!.fact, attrs[0]!.confidence)
  }

  return { schema: field, value: null, status: "needs-input" }
}

function finish(
  field: FormFieldSchema,
  fact: ProfileFact | undefined,
  confidence: Confidence,
): MappingResult {
  if (!fact) return { schema: field, value: null, status: "needs-input" }

  // THE GATE. The only place a value becomes fillable.
  if (fact.confidence < REVIEW_THRESHOLD) {
    return { schema: field, value: null, status: "needs-review" }
  }

  const value = coerceToKind(fact.value, field)

  // A coercion failure must degrade to needs-input, not pass through a broken value.
  if (value === null) {
    return { schema: field, value: null, status: "needs-input" }
  }

  return {
    schema: field,
    value: {
      value,
      // Take the MINIMUM of fact confidence and rule confidence.
      // A perfect label match cannot rescue a fact we are unsure about.
      confidence: {
        score: Math.min(fact.confidence, confidence.score),
        reason: confidence.reason,
      },
      provenance: fact.provenance,
      editedByUser: false,
    },
    status: "ready",
  }
}
```

> **`Math.min` on the two confidences, and it is not obvious.** Layer 1 gives you `1.0` for
> an exact label match, but if the *stored fact* is only 0.6 confident, filling it confidently
> is a lie. The confidence of an answer is the confidence of its weakest link. If you take
> `max` you will ship confidently-wrong fills.

### Coercion by field kind

```ts
export function coerceToKind(value: string, field: FormFieldSchema): string | null {
  switch (field.kind) {
    case "number": {
      const n = toNumber(value)
      if (n === null) return null
      // The label may demand a scale the fact does not carry.
      const m = field.label.match(/out of\s*(\d+(?:\.\d+)?)/i)
      if (m) return toCGPA(value, Number(m[1])) ?? null
      return String(n)
    }
    case "date":
      return toISODate(value)

    case "select": {
      if (!field.options?.length) return value
      const v = value.trim().toLowerCase()
      const hit = field.options.find(
        (o) => o.toLowerCase() === v
          || o.toLowerCase().includes(v)
          || v.includes(o.toLowerCase()),
      )
      // A free-text value in a closed list is a bug, not a best guess.
      return hit ?? null
    }

    case "radio":
    case "checkbox":
      return /^(y|yes|true|1|married|male)$/i.test(value.trim()) ? "true" : "false"

    case "file":
      // §11: never fill a file input. Refuse at the type level.
      return null

    default:
      return value.trim() || null
  }
}
```

**This function is the difference between "plausible" and "correct."** `field: "select"`
returning `null` when the value is not an option is doing something important: it is refusing
to type "Bengaluru" into a dropdown that contains only `["Bangalore", "Mysore"]`. A fuzzy
match that picks the wrong state on a government form is a rejected application.

---

## Step 5 — Test the moat

```ts
// packages/mapping/src/resolve.test.ts
import { describe, it, expect } from "vitest"
import { resolveField, resolveAll } from "./resolve"
import { buildAliasIndex } from "./normalize"
import aliases from "../data/aliases.json"

const profile = makeProfile({
  fullName:      fact("Rohit Jadhav", "profile", 1.0),
  email:         fact("r@example.com", "profile", 1.0),
  phone:         fact("+91 98765 43210", "profile", 1.0),
  tenthPercentage: fact("87.4%", "document", 0.98),
  cgpa:          fact("8.7", "document", 0.96),
})

const idx = buildAliasIndex(aliases)
const facts = indexFacts(profile)

const check = (label: string) =>
  resolveField(field(label), profile, facts, idx, aliases)

describe("layer 1 — exact", () => {
  it("maps a plain label", () => {
    expect(check("Email address").value?.value).toBe("r@example.com")
  })
})

describe("layer 2 — containment", () => {
  it("strips noise", () => {
    expect(check("* Full Name of the Applicant (Required)").value?.value).toBe("Rohit Jadhav")
  })
  it("scores containment below exact", () => {
    expect(check("Please enter your email address here").confidence.score)
      .toBeLessThan(1.0)
  })
})

describe("the hard cases", () => {
  it("refuses to guess DD/MM when ambiguous", () => {
    const r = check("Date of Birth")
    // Must be either a valid ISO date or needs-input. Never a mangled string.
    if (r.value) expect(r.value.value).toMatch(/^\d{4}-\d{2}-\d{2}$/)
    else expect(r.status).toBe("needs-input")
  })

  it("scales CGPA to the label's demand", () => {
    const f = field("CGPA out of 4")
    expect(coerceToKind("8.7", f)).toBe("3.48")
  })

  it("refuses a value absent from a closed option list", () => {
    const f = { ...field("State"), kind: "select" as const, options: ["Maharashtra", "Kerala"] }
    expect(coerceToKind("Bengaluru", f)).toBeNull()
  })

  it("never proposes a file upload", () => {
    expect(coerceToKind("marksheet.pdf", { ...field("Marksheet"), kind: "file" })).toBeNull()
  })

  it("takes the MINIMUM of fact and rule confidence", () => {
    const weak = makeProfile({ email: fact("r@example.com", "inference", 0.6) })
    const r = resolveField(field("Email address"), weak, indexFacts(weak), idx, aliases)
    expect(r.value).toBeNull()
    expect(r.status).toBe("needs-review")
  })
})
```

```bash
pnpm --filter @refrain/mapping test
```

**Then run it against your 40 saved real forms** and print the score. That number is your
metric:

```ts
// packages/mapping/scripts/score.ts
const total = schemas.reduce((n, s) => n + s.fields.length, 0)
const resolved = results.filter((r) => r.status === "ready").length
const correct  = results.filter((r) => r.status === "ready" && r.value?.editedByUser === false).length
console.log(`auto-filled ${resolved}/${total} (${((resolved/total)*100).toFixed(0)}%)`)
```

Track it. Every time you add 20 aliases, the number should go up and no test should break. If
it goes down, you added an over-broad alias — a `contains` rule like `"name"` that swallows
`username`, `father_name`, and `institution_name`. That is the most common regression in this
system and it is invisible without the score script.

---

## Step 6 — Local inference (the fallback layer)

Only for what rules cannot do. Budget: **under 8% of fields**.

### 6a — Feature detection, in the right order

```ts
// packages/mapping/src/inference/engine.ts

export type Engine = "chrome-prompt" | "ollama" | "none"

export async function detectEngine(): Promise<Engine> {
  // 1. Chrome's built-in on-device model. Free, private, already on the machine.
  const L = (globalThis as { LanguageModel?: any }).LanguageModel
  if (L?.availability) {
    try {
      const avail = await L.availability()
      if (avail === "available" || avail === "downloadable") return "chrome-prompt"
    } catch { /* fall through */ }
  }

  // 2. Ollama, if the user runs it. Explicit, visible, user-controlled.
  try {
    const res = await fetch("http://localhost:11434/api/tags", { signal: AbortSignal.timeout(800) })
    if (res.ok) return "ollama"
  } catch { /* not running */ }

  return "none"
}
```

> **Never put `langchain`, `transformers.js`, or any model download in the extension bundle.**
> A shipped model is 500MB+ and turns a 3MB extension into a 500MB extension. Chrome's
> `LanguageModel` downloads the model itself, managed by the browser, cached outside your
> bundle. That is why it is the first choice.

### 6b — The one thing a model is allowed to do

```ts
// packages/mapping/src/inference/classify.ts

const SYSTEM = `You map web form labels to a fixed set of profile keys.

Rules:
- Reply with ONLY a JSON array. No prose, no markdown fences.
- Reply "unknown" if no key fits. Guessing is worse than abstaining.
- Reply "ambiguous:A,B" if two keys are equally plausible.

Keys: ${Object.keys(aliases).join(", ")}`

export async function classifyLocally(
  labels: string[],
  candidates: string[],
): Promise<Map<string, string>> {
  const out = new Map<string, string>()
  if (!labels.length) return out

  const L = (globalThis as any).LanguageModel
  if (!L?.create) return out

  try {
    const session = await L.create({
      initialPrompts: [{ role: "system", content: SYSTEM }],
      temperature: 0,          // determinism matters more than creativity here
      topK: 1,
    })
    const prompt = `Labels:\n${labels.map((l, i) => `${i}. ${l}`).join("\n")}`
    const raw = await session.prompt(prompt, { outputLanguage: "en" })

    // Models wrap JSON in prose and fences no matter what you ask. Parse defensively.
    const json = raw.match(/\[[\s\S]*\]/)?.[0]
    if (!json) return out

    for (const [i, key] of (JSON.parse(json) as string[]).entries()) {
      if (key && key !== "unknown" && !key.startsWith("ambiguous")) {
        const label = labels[i]
        if (label) out.set(label, key)
      }
    }
  } catch {
    // No model, no download, an error. The rules layer already ran.
    return out
  }
  return out
}
```

**Three decisions worth copying:**

**`temperature: 0, topK: 1`.** Mapping is not creative writing. You want the same label to
map to the same key every time, or your review screen is unstable and users stop trusting it.

**The model is allowed to abstain.** `"unknown"` is a first-class output. A model that always
answers is a model that always guesses, and you have already decided guessing is worse.

**It runs last, on the leftovers only.** Rules first, model on the ≤8% rules could not
resolve. Local means ₹0, but slow and wrong are still wrong.

### 6c — Ollama fallback

```ts
async function classifyViaOllama(labels: string[], candidates: string[]) {
  const res = await fetch("http://localhost:11434/api/chat", {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify({
      model: "llama3.2:3b",        // 3B is plenty for label mapping
      stream: false,
      format: "json",               // Ollama's grammar-constrained JSON
      messages: [
        { role: "system", content: SYSTEM },
        { role: "user", content: labels.join("\n") },
      ],
    }),
  })
  const { message } = await res.json()
  return JSON.parse(message.content)
}
```

> **`format: "json"` makes Ollama structurally unable to emit prose.** A constraint beats a
> prompt asking nicely. Apply the same idea to the Chrome path: parse with a regex that
> *only* accepts a JSON array, and discard anything else.

**Never add a cloud provider.** Not "later," not "for users without a local model." §11's
local-first claim is the moat, and a single `fetch("https://api.openai.com/...")` in the tree
destroys it. If you ever want BYOK, it is a §19 decision, it is opt-in per user, and it is
labelled loudly in the UI. Not in Phase 1.

---

## Step 7 — Commit and record the decision

```bash
git add -A
git commit -m "feat(mapping): 3 rule layers + coercion + local inference fallback, 300 aliases"
```

Add these two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | Rules first, local model on ≤8% leftovers | ₹0 per fill, instant, and each answer carries a reason a user can read. The model is a fallback, not the engine. |
| 2026-10-XX | Confidence = min(fact, rule) | A perfect label match cannot rescue a fact we are unsure about. |
| 2026-10-XX | `null` coercion always degrades to `needs-input` | Three characters typed by the user beats a wrong number on a scholarship form. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **`resolveField` and the min-confidence gate** | **You.** It is the gate. Own it |
| `normalizeLabel` and its NOISE list | **You** — you will add twenty entries after using it |
| `toISODate`, `toCGPA`, `toNumber` | **You.** Date logic is where silent damage happens |
| **`aliases.json` — every single entry** | **You. 100%.** This is the moat. This is what OpenCode cannot do |
| The test suite | **You** for the hard cases, **OpenCode** for the 40 mechanical ones |
| `coerceToKind` | **You** |
| `detectEngine` / Ollama call | **OpenCode** |
| `classifyLocally` prompt engineering | **OpenCode**, then you rewrite the prompt by hand and test it |
| `score.ts` regression script | **OpenCode** |
| Extracting candidate labels from the 40 saved forms | **OpenCode** — it is mechanical |

---

## Gotchas in this chapter

**"I added `name` as an alias and now every field maps to full name."** Containment matches
substring. `institution_name`, `father_name`, `username`, `bank_name` all contain `name`.
**Fix: never put a bare `name` alias in the dataset.** Use `full name`, `applicant name`,
`your name`. Grep your alias file for 3-character entries and delete them.

**Confidence is 1.0 on everything.** You are returning the *rule* score and ignoring the fact's.
`Math.min` is the fix. Write a test for it.

**Your select fields always fail.** `coerceToKind` returns `null` when the value is not in
`options`. That is correct behaviour and it is a *data* problem: your profile stores
`"Maharashtra"` and the dropdown says `"Maharashtra State"`. Extend `field.options` matching
or add a state-name alias table. Do not loosen the check.

**The model returns a string, not JSON.** Every model does this at least once. Parse with the
regex, discard on failure, and never `JSON.parse` unguarded — an uncaught throw inside
`onMessage` kills the handler and the panel hangs.

**`fetch` to `localhost:11434` throws a CORS error in the panel.** Extension pages need host
permission for that origin, and you will see it as a console error rather than a thrown
exception. Catch and treat as "engine unavailable" — which it is.

**Scores went up and something got worse.** Almost always an over-broad alias. The score
script tells you how many fields resolve; it cannot tell you the 3 that now resolve wrongly.
**Read the diff in `score.ts --verbose`, not just the number.**

**A user reports "it filled my name into the college name field."** Two possibilities: an
alias collision, or `resolveLabel` returned a container's text instead of the field's. Log the
raw label and the normalised label on every mapping during a debug session.

---

## Verify before moving on

- [ ] Every one of the 40 saved real forms resolves at a high rate
- [ ] `score.ts` reports a number, and you wrote it down
- [ ] Low-confidence facts are **never** filled, even on an exact label match
- [ ] CGPA rescaling works for `out of 4` and `out of 10`
- [ ] A value missing from a closed option list yields `needs-input`, never a near-match
- [ ] `kind: "file"` is refused before it reaches the page
- [ ] Ambiguous fields report `needs-review` and name both candidates
- [ ] `detectEngine` finds Chrome or Ollama, and degrades cleanly to `none`
- [ ] The model output path survives prose-wrapped, fenced, and malformed JSON
- [ ] `aliases.json` has ≥300 entries and **zero** 3-character aliases
- [ ] `pnpm --filter @refrain/mapping test` passes

---

## Check yourself before Chapter 15

1. **Why is `Math.min` on the two confidences correct, and what does `Math.max` break?**
2. **Why must a coercion failure return `null` rather than the original value?**
3. **What does `ambiguous:A,B` protect the user from that `A` alone would not?**
4. **Why does `temperature: 0` matter more here than for any other AI feature you will build?**
5. **An alias of `"date"` is fine but `"name"` is not. Explain the asymmetry.**
6. **Why is the alias dataset the moat rather than the resolver code?**
7. **Your score went 71% → 83%. How do you find the three fields that got worse?**

---

**Next: [Chapter 15 — Fermata](./15-fermata-review-screen.md)** — the hero screen. Provenance
on every row, inline editing, the missing-info path, XState, and the moment the user presses
submit and Refrain does not.