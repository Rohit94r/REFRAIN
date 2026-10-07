# Chapter 2 — The Monorepo

> **Day 2 · Goal:** `pnpm dev` **boots all three surfaces at once.**
>
> One command, six packages, two apps. You are done when a single terminal starts
> everything and one broken package fails loudly instead of silently.

---

## Understand this first

### Why a monorepo and not three repos

Your side panel and your web app are **the same React application with two different entry
points.** They share the mascot, the design tokens, the vault, the field schemas, and the
mapping engine. That is not a preference — it is the reason you can have a proper web UI
at all (§6.2).

Three separate repos would mean:

- Publishing six internal packages to a private registry just to import them
- Seven version bumps every time the field schema changes
- Breaking changes landing in the web app an hour before you want them

A monorepo makes shared code a **local file path.** The dependency graph is enforced by
TypeScript, the build order by Turborepo, and the cost of a mistake is a failed typecheck
instead of a runtime incident.

### Three programs, two apps — resolving the apparent contradiction

Chapter 1 listed three programs. This chapter creates two apps. Both are correct, and the
difference is the single most confusing thing about WXT:

| Program        | Built by              | Entry point file                                                |
| -------------- | --------------------- | ---------------------------------------------------------------- |
| Web app        | `apps/web`            | `src/main.tsx`                                                   |
| Side panel     | `apps/extension`      | `entrypoints/sidepanel/App.tsx`                                  |
| Content script | `apps/extension`      | `entrypoints/content/index.ts`                                  |

**WXT models extension entry points as files inside one app, not as apps.** So the extension
has two entry points and therefore one build. Three things ship; two packages produce them.

### The folder structure

```
refrain/
├── apps/                    ← the two things that actually ship
│   ├── web/                 the full web app (profile, documents, setlist, settings)
│   └── extension/           MV3 — WXT. Contains BOTH extension entry points:
│       ├── sidepanel/           entrypoints/sidepanel/  the companion UI
│       └── content/             entrypoints/content/    the field scanner
│
└── packages/                ← the things that get imported
    ├── fields/              canonical Zod schemas — zero dependencies
    ├── vault/               encrypted IndexedDB profile + documents
    ├── mapping/             form schema → profile field resolution
    ├── extract/             pdf.js + Tesseract document parsing
    ├── ui/                  design system, mascot, review components
    └── tsconfig/            shared TypeScript presets
```

`apps/` **are things with an entry point and a build output.** `packages/` **are libraries.**
That is the whole distinction. A library has no `main` script.

> **There is no `apps/sidepanel`, and that is a WXT decision rather than an omission.**
> WXT models extension entry points as *files* inside one app. So the side panel is
> `apps/extension/entrypoints/sidepanel/App.tsx`, built by the same Vite pipeline as the content
> script, with the same React, the same Tailwind tokens, and the same `@refrain/*` imports.
>
> Giving the side panel its own Vite app would mean **two build systems for one UI**, and the
> promise that the side panel and the web app are "the same app with two doors" would be a lie
> the first time a token or a component drifted between them. Two apps, two build outputs: `web`
> and `extension`. Chapter 7 builds the side panel as an entry point.

### `packages/fields` is an addition to the master document

`refrain.md` §10 lists only `vault`, `mapping`, `extract`, and `ui`. This map adds `fields`,
because §10 also says:

> *The mapping engine should share Zod schemas with the form layer. One definition of "a field"
> used by both the UI and the resolver. Avoids a whole class of drift bugs.*

If that schema lives in `mapping`, then `ui` must depend on `mapping` — a UI library depending
on an engine. Wrong direction. `fields` is that definition with **zero dependencies**, and
both `ui` and `mapping` depend on it. That is the requirement from §10 made concrete.

> This is a deliberate deviation. **Record it as a §19 decision when you finish Chapter 8**,
> at the moment you feel the benefit. Documentation written from theory rots.

### Why `workspace:*` and not a version number

```json
{ "dependencies": { "@refrain/ui": "workspace:*" } }
```

`workspace:*` means "the version in the monorepo, right now, linked locally." pnpm never
tries to download it. When you ship, you replace it with a real version.

This is why pnpm's strictness matters: **you cannot import a package you did not declare.**
You will hit this in Chapter 3 and it will look like a bug. It is a feature.

---

## Step 1 — The root

```bash
mkdir -p refrain && cd refrain
git init
code .          # or your editor
```

> **Where this lives matters.** Put `refrain/` **outside** `refrain-buildmap/`. They are two
> different things: one is the curriculum, one is the thing the curriculum builds. If you nest
> the monorepo inside its own instructions, every `git status` in Chapter 4 is confusing and
> you will eventually commit `node_modules` into the docs repo.
>
> ```bash
> cd ~/projects          # one level up from the map
> mkdir -p refrain && cd refrain
> ```
>
> Then `~/projects/refrain/` is the monorepo and `~/projects/reprise/refrain-buildmap/` is what
> you are reading.

### `package.json` (root)

```json
{
  "name": "refrain",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@10.32.1",
  "engines": { "node": ">=22" },
  "scripts": {
    "dev": "turbo dev",
    "build": "turbo build",
    "typecheck": "turbo typecheck",
    "lint": "turbo lint",
    "test": "turbo test",
    "check": "pnpm typecheck && pnpm lint && pnpm test && pnpm build",
    "clean": "turbo clean && rm -rf node_modules"
  },
  "devDependencies": {
    "turbo": "^2.5.0",
    "typescript": "^5.9.0",
    "prettier": "^3.6.0",
    "eslint": "^9.0.0"
  }
}
```

`pnpm check` **is the command you will run most.** It chains typecheck → lint → test → build
in the order where the cheapest, most-likely-to-fail check runs first. Put that in your
muscle memory now.

**Pin** `packageManager`. It tells Corepack which pnpm version your lockfile expects. Without
it, your teammate and CI can silently use different pnpm and produce different lockfiles.

> **Verify Turbo actually installed, and check the major version.** This matters because
> Turbo 1.x and 2.x use a different key name for the same concept, and it is the first thing
> that will bite you:
>
> ```bash
> pnpm install
> pnpm exec turbo --version
> # → turbo 2.11.7  (or similar 2.x)
> ```
>
> If you see `1.x`, pin `"turbo": "^2.0.0"`. The rest of this chapter assumes 2.x.

### `pnpm-workspace.yaml`

```yaml
packages:
  - "apps/*"
  - "packages/*"

# pnpm 10 blocks lifecycle scripts by default. Allowlist the ones we actually need.
onlyBuiltDependencies:
  - esbuild
  - sharp
  - unrs-resolver
```

> **This block is the pnpm 10 change that will confuse every tutorial you find.** pnpm 10 no
> longer runs third-party `postinstall` scripts unless you allowlist them. This is a security
> feature. Without it, `esbuild` (which Vite depends on) fails to install its binary and you
> get a confusing "esbuild not found" hours later. If you add a dependency that needs a build
> step and it mysteriously fails, this list is why.
>
> **Keep the list short.** Chapter 18 makes this a security posture: every entry is code that
> runs with your credentials at install time. Three entries is a deliberate choice. A list of
> twenty is a supply-chain risk you did not decide on purpose.

### `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": ["$TURBO_DEFAULT$", "!**/*.md"],
      "outputs": ["dist/**", ".output/**", ".wxt/**"]
    },
    "typecheck": { "dependsOn": ["^build"], "outputs": [] },
    "lint": { "outputs": [] },
    "test": { "dependsOn": ["^build"], "outputs": [] },
    "dev": { "cache": false, "persistent": true },
    "clean": { "cache": false }
  }
}
```

| Line                      | What it does, and why you need it                                                                                                                                                 |
| ------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `"tasks"`                 | **Turborepo 2.x renamed** `pipeline` **→** `tasks`.** Every tutorial you find shows `pipeline`. If your tasks are not running, this is 90% of why.                                        |
| `"dependsOn": ["^build"]` | The `^` means *upstream dependencies first*. `mapping` builds before the extension that imports it. Without it you get "cannot find module" half the time.                                |
| `"inputs"`                | What counts as a cache key. `$TURBO_DEFAULT$` is "all git-tracked files". Without it, Turbo caches on its own guess and replays stale builds.                                     |
| `"persistent": true`      | Marks `dev` as a long-running process. Without it Turborepo waits for it to exit and hangs forever.                                                                               |
| `"cache": false` on dev   | Caching a watch process is nonsense.                                                                                                                                              |
| `"outputs"`               | What Turborepo is allowed to cache. If you omit it, builds are cached but the output is not restored and nothing works.                                                            |

> **The `pipeline` → `tasks` rename produces a real, loud failure — verified on Turbo 2.11:**
>
> ```
> x Found `pipeline` field instead of `tasks`.
>    ,-[turbo.json:1:15]
>  1 | { "pipeline": { "build": { ... } } }
>    :               ^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^
>    |                                               `-- Rename `pipeline` field to `tasks`
>    `----
> help: Changed in 2.0: `pipeline` has been renamed to `tasks`.
> ```
>
> The exit code is `1` and **no task runs at all.** So if `pnpm build` "does nothing, no
> errors, no output," this is why. The chapter's earlier draft said this fails *silently* —
> it does not; Turbo 2.11 fails loudly, which is better. What is silent is the *old* habit of
> copying a 1.x tutorial.

### `.gitignore`

```gitignore
node_modules/
dist/
.output/
.wxt/
.turbo/
.env
.env.local
*.log
.DS_Store

# Never commit anything the user uploaded
*.pdf
*.png
*.jpg
```

> Those last three lines matter more than they look. You are building a tool that holds
> marksheets, photographs, and signatures. A stray `git add .` that commits a test marksheet
> is a §11 privacy violation with your own git history as evidence. Put them in **before** you
> write the extractor, not after.
>
> **Chapter 17 adds one more line you will need later:**
> `apps/extension/.env` — the extension's dev secrets must never be committed either.

### `tsconfig.base.json` (root — do not build from this again)

```json
{
  "$schema": "https://json.schemastore.org/tsconfig",
  "compilerOptions": {
    "target": "ES2022",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "module": "ESNext",
    "moduleResolution": "bundler",
    "moduleDetection": "force",
    "jsx": "react-jsx",
    "strict": true,
    "noUncheckedIndexedAccess": true,
    "noImplicitOverride": true,
    "noFallthroughCasesInSwitch": true,
    "verbatimModuleSyntax": true,
    "isolatedModules": true,
    "resolveJsonModule": true,
    "skipLibCheck": true,
    "forceConsistentCasingInFileNames": true,
    "noEmit": true
  }
}
```

> **Why the root file says "do not build from this again."** In Chapter 5 you add `paths`
> aliases for the web app, and in Chapter 7 you find that the extension needs different `lib`
> settings than the web. If every package copies this file and edits it, you will have eight
> subtly different `strict` setups and no way to tell them apart. The next step is the fix.

---

## Step 2 — `packages/tsconfig` (shared presets)

You will have seven packages. None of them should redefine `strict: true`.

```
packages/tsconfig/
├── package.json
├── base.json
├── react-library.json
└── app.json
```

`base.json` extends the root, plus library defaults:

```json
{
  "extends": "../../tsconfig.base.json",
  "compilerOptions": {
    "declaration": true,
    "composite": true
  }
}
```

`react-library.json`:

```json
{
  "extends": "./base.json",
  "compilerOptions": { "jsx": "react-jsx" }
}
```

`app.json`:

```json
{
  "extends": "./base.json",
  "compilerOptions": {
    "noEmit": true,
    "types": ["vite/client"]
  }
}
```

`packages/tsconfig/package.json`:

```json
{
  "name": "@refrain/tsconfig",
  "version": "0.0.0",
  "private": true,
  "files": ["base.json", "react-library.json", "app.json"]
}
```

Every package's `tsconfig.json` is now four lines:

```json
{ "extends": "@refrain/tsconfig/react-library.json" }
```

> **`packages/tsconfig` has no scripts at all**, which is deliberate. It ships configuration
> files, not code, so it should never be typechecked, built, or cached as a task. When you run
> `pnpm typecheck` and count the packages that report, **this one will not appear.** That is
> correct, not a bug.

---

## Step 3 — `packages/fields` (start here — everything depends on it)

The canonical definition of "a field." **Zero dependencies.** This is the contract between
your UI and your mapping engine.

```bash
mkdir -p packages/fields/src && cd packages/fields
```

`package.json`:

```json
{
  "name": "@refrain/fields",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "exports": { ".": "./src/index.ts" },
  "scripts": {
    "typecheck": "tsc --noEmit",
    "lint": "eslint src",
    "test": "vitest run",
    "build": "tsc --noEmit",
    "clean": "rm -rf dist .turbo"
  },
  "dependencies": { "zod": "^4.0.0" },
  "devDependencies": {
    "typescript": "^5.9.0",
    "vitest": "^3.0.0"
  }
}
```

`tsconfig.json`:

```json
{
  "extends": "@refrain/tsconfig/react-library.json",
  "include": ["src"]
}
```

> **Note** `"exports": { ".": "./src/index.ts" }` **— you are exporting source, not a build
> artifact.** In a private monorepo this is simpler and faster than compiling every package.
> Vite and Turborepo both handle it. If you ever need a real build, you flip this and add
> `dist/` to `exports`. Do not build the whole toolchain twice.
>
> **Note** `"build": "tsc --noEmit"` **looks like a no-op and is intentional.** Turborepo needs
> every package to have a `build` script so `dependsOn: ["^build"]` has something to order.
> This one typechecks instead of emitting, because there is nothing to emit. Keep it —
> removing the script breaks the graph silently.

### `src/index.ts`

```ts
import { z } from "zod"

/* ── Confidence ──────────────────────────────────────────────── */

export const Confidence = z.object({
  /** 0–1. Anything below REVIEW_THRESHOLD must never be auto-filled. */
  score: z.number().min(0).max(1),
  /** Why this number. Shown in the review screen. Never a bare float. */
  reason: z.string(),
})
export type Confidence = z.infer<typeof Confidence>

/**
 * The one number that decides whether Refrain fills or asks.
 * A resolver may NEVER hardcode a different threshold — that is how
 * "it filled the wrong value without asking" bugs happen.
 */
export const REVIEW_THRESHOLD = 0.75

/* ── Provenance ──────────────────────────────────────────────── */

export const SourceKind = z.enum(["profile", "document", "verse", "manual", "inference"])
export type SourceKind = z.infer<typeof SourceKind>

/**
 * Pillar 1: every fact carries where it came from and when.
 * Provenance is what makes the review screen trustworthy — if it is
 * optional, it will be skipped, and then nothing is trustworthy.
 */
export const Provenance = z.object({
  kind: SourceKind,
  /** Human-readable: "marksheet.pdf", "you", "your internship verse" */
  label: z.string(),
  /** Exact path into the profile graph, e.g. "education[0].cgpa" */
  path: z.string().optional(),
  extractedAt: z.string().datetime(),
  /** Document page or sheet number, when kind === "document" */
  locator: z.string().optional(),
})
export type Provenance = z.infer<typeof Provenance>

/* ── The canonical field ─────────────────────────────────────── */

export const FillKind = z.enum(["text", "number", "email", "tel", "url", "date", "textarea", "select", "checkbox", "radio", "file"])
export type FillKind = z.infer<typeof FillKind>

export const FieldValue = z.object({
  /**
   * ALWAYS a string. An <input> only accepts a string; anything else
   * is a silent-empty-submit bug. Format upstream, not here.
   */
  value: z.string(),
  confidence: Confidence,
  provenance: Provenance,
  /** Set when the user's correction overrode the machine's answer. */
  editedByUser: z.boolean().default(false),
})
export type FieldValue = z.infer<typeof FieldValue>

/* ── Form schema (what the content script returns) ───────────── */

export const FormFieldSchema = z
  .object({
    /** Stable within one page load. Used as the React key. */
    id: z.string(),
    label: z.string(),
    name: z.string().optional(),
    kind: FillKind,
    required: z.boolean(),
    options: z.array(z.string()).optional(),
    /** True when a visible input is driven by a paired hidden input. */
    hasHiddenDriver: z.boolean().default(false),
    /** Set when the field is in an iframe — you need the frame id to write back. */
    frameId: z.number().optional(),
  })
  /**
   * `.strict()` on the FIELD, not on the form.
   *
   * Zod strips unknown keys by default, which is correct for a wire
   * protocol (Chapter 17) but WRONG here: a content script that
   * accidentally includes `value: 87.4` would have it silently
   * dropped, and you would debug a missing field instead of a typo.
   * Unknown keys on a field are always a bug, so reject them loudly.
   */
  .strict()
export type FormFieldSchema = z.infer<typeof FormFieldSchema>

export const FormSchema = z.object({
  url: z.string(),
  title: z.string(),
  fields: z.array(FormFieldSchema),
  /** Hard-blocked by the §11 "no" list: gov KYC, competitive exams. */
  blocked: z.boolean().default(false),
  blockedReason: z.string().optional(),
  captchaDetected: z.boolean().default(false),
})
export type FormSchema = z.infer<typeof FormSchema>

/* ── The mapping result (what the resolver returns) ──────────── */

export const MappingResult = z.object({
  schema: FormFieldSchema,
  /** null means "we could not map this — ask the user." Never guess. */
  value: FieldValue.nullable(),
  status: z.enum(["ready", "needs-review", "needs-input", "unsupported"]),
})
export type MappingResult = z.infer<typeof MappingResult>

/* ── Suffix injection for IDs ───────────────────────────────── */

export const withSuffix = (id: string, suffix: string): string => `${id}__${suffix}`
```

> **`FormSchema` is deliberately NOT `.strict()`, and the asymmetry with `FormFieldSchema` is
> the point.** Chapter 17's rule is "strip unknown keys on every message boundary, because you
> cannot force a shipped extension to update." That rule is about *adding* fields over time.
> A field object is different: a key you do not recognise there is a typo or a bug, not a
> future version, because the field's identity is its label and its kind, not a growing bag of
> properties. Strict where a mistake is silent, lenient where a mistake is only "older version."
>
> **Why `withSuffix` exists at all.** Chapter 7 has to write the same field twice when a form
> lives in an iframe — once in the top frame's ID space and once in the child's. `email` becomes
> `email__frame3`. A helper means that convention is in one place instead of in four
> call sites that each concatenate a slightly different string.

### Why these seven decisions matter

| Decision                               | What it prevents                                                                                                    |
| -------------------------------------- | ------------------------------------------------------------------------------------------------------------------- |
| `value` is always `string`             | Chapter 7's silent-empty-submit bug. This is the single most important line in the file.                            |
| `confidence` requires a `reason`       | You will never be able to show "why" on the review screen. There is no bare-float fallback.                         |
| `provenance` is required, not optional | Optional means skipped. Skipped means the review screen lies.                                                       |
| `editedByUser`                         | This is your **Phase 1 success metric** — correction rate must *decrease*. You cannot measure it without this flag. |
| `REVIEW_THRESHOLD` is one constant     | One place to change the bar. Never a magic number in a resolver.                                                    |
| `blocked` on the form                  | §11's hard "no" list is enforced in the **schema**, so it is impossible to forget at the call site.                  |
| `frameId`                              | Chapter 7's iframe gotcha, solved in the type instead of in a bug report.                                           |

### `src/index.test.ts` — write this yourself

```ts
import { describe, it, expect } from "vitest"
import { FormSchema, FormFieldSchema, FieldValue, withSuffix } from "./index"

describe("FormFieldSchema", () => {
  it("rejects a number where a value would go", () => {
    // Complete and valid EXCEPT for the stray `value` key, which is
    // exactly the mistake we want `.strict()` to catch.
    const bad = {
      id: "1",
      label: "Name",
      kind: "text",
      required: false,
      value: 87.4,
    }
    const r = FormFieldSchema.safeParse(bad)
    expect(r.success).toBe(false)
    if (!r.success) {
      expect(r.error.issues[0]?.code).toBe("unrecognized_keys")
      expect(r.error.issues[0]?.keys).toContain("value")
    }
  })

  it("requires a complete field shape", () => {
    const r = FormFieldSchema.safeParse({ id: "1", label: "Name" })
    expect(r.success).toBe(false)
  })
})

describe("FormSchema", () => {
  it("rejects a field with no id", () => {
    const r = FormSchema.safeParse({ url: "u", title: "t", fields: [{ value: 87.4 }] })
    expect(r.success).toBe(false)
  })

  it("applies the hasHiddenDriver default", () => {
    const r = FormSchema.safeParse({
      url: "u",
      title: "t",
      fields: [{ id: "1", label: "N", kind: "text", required: false }],
    })
    expect(r.success).toBe(true)
    if (r.success) expect(r.data.fields[0]?.hasHiddenDriver).toBe(false)
  })
})

describe("FieldValue", () => {
  it("rejects a numeric value — this is the Chapter 7 bug, caught at the schema", () => {
    const r = FieldValue.safeParse({
      value: 87.4,
      confidence: { score: 0.98, reason: "10th percentage, marksheet p.3" },
      provenance: {
        kind: "document",
        label: "marksheet.pdf",
        extractedAt: "2026-10-01T10:00:00.000Z",
      },
    })
    expect(r.success).toBe(false)
  })

  it("defaults editedByUser to false", () => {
    const r = FieldValue.safeParse({
      value: "87.4",
      confidence: { score: 0.98, reason: "10th percentage" },
      provenance: {
        kind: "document",
        label: "marksheet.pdf",
        extractedAt: "2026-10-01T10:00:00.000Z",
      },
    })
    expect(r.success).toBe(true)
    if (r.success) expect(r.data.editedByUser).toBe(false)
  })

  it("rejects a confidence score above 1", () => {
    const r = FieldValue.safeParse({
      value: "x",
      confidence: { score: 1.4, reason: "overconfident" },
      provenance: { kind: "manual", label: "you", extractedAt: "2026-10-01T10:00:00.000Z" },
    })
    expect(r.success).toBe(false)
  })
})

describe("withSuffix", () => {
  it("is deterministic", () => {
    expect(withSuffix("email", "frame3")).toBe("email__frame3")
  })
})
```

**Run it:** `pnpm test`

**What you should see** (verified against zod 4.6.5 / vitest 3):

```
✓ src/index.test.ts (7 tests) 2ms
 Test Files  1 passed (1)
      Tests  7 passed (7)
```

Then deliberately break one test and read the failure. Change `withSuffix("email", "frame3")`
to expect `email-frame3` and watch the diff.

> **Why the first test is stricter than the version in earlier drafts of this chapter.** The
> original test was:
>
> ```ts
> expect(FormSchema.safeParse({ url:"u", title:"t", fields:[{ value: 87.4 }] }).success).toBe(false)
> ```
>
> **That test passes for the wrong reason, and it is worth understanding why.** It fails
> because `id`, `label`, `kind`, and `required` are all missing — not because of the number. The
> assertion `success === false` is satisfied by any of four unrelated violations, so if you
> deleted the `value: 87.4` entirely the test would still pass.
>
> Worse: because Zod **strips** unknown keys rather than rejecting them, a *complete* field with
> a stray `value: 87.4` would **parse successfully** and the number would vanish:
>
> ```
> parse success?  true
> resulting field: {"id":"1","label":"Name","kind":"text","required":false,"hasHiddenDriver":false}
> ```
>
> That is the silent-empty-field bug, at the schema layer, and a loose test would never have
> told you. **A test that passes for the wrong reason is worse than no test**, because it
> removes the pressure to look. This is why the fixed version asserts on `issues[0].code` and
> `keys` — it proves *which* rule fired.

> **This is the moment to start using OpenCode.** Ask it:
> *"Write 8 more adversarial Zod tests for* `@refrain/fields` *— focus on partial objects,
> wrong enum values, and missing required keys. Do not change the source."*
> Reading its output and understanding why each test exists **is** the 40%.
>
> **Then apply the same audit to what it gives you.** For each test it writes, ask: *what
> exactly would fail if this rule were deleted from the schema?* If you cannot answer, the test
> is the loose kind above and you should rewrite it.

---

## Step 4 — `packages/vault`, `mapping`, `extract`, `ui`

Create them with the identical shape. Do the mechanical work by hand **once**, then you
understand the template forever.

```bash
for p in vault mapping extract ui; do
  mkdir -p packages/$p/src
done
```

Each gets a `package.json` like `fields`, changing only `name` and `exports`. Dependencies:

| Package   | Depends on                                      | Why                                                            |
| --------- | ----------------------------------------------- | -------------------------------------------------------------- |
| `vault`   | `dexie`, `@refrain/fields`                      | It stores field-typed documents                                |
| `mapping` | `@refrain/fields`, `zod`                        | Pure logic, no UI, no DOM                                      |
| `extract` | `pdfjs-dist`, `tesseract.js`, `@refrain/fields` | Document → field values                                        |
| `ui`      | `react`, `@refrain/fields`                      | Components only. **Never depends on** `mapping` **or** `vault` |

> **The dependency rule, memorise it:** `fields` ← everyone. `ui` depends on `fields` only.
> `mapping` depends on `fields` only. `vault` depends on `fields` only. The apps wire them
> together. **Packages must not import each other sideways.** If `vault` needs `mapping`, you
> have put business logic in the wrong box.
>
> **`mapping` needs `zod` because it validates resolver output against `MappingResult`.** It does
> not re-declare the field shape; it imports it. That is the whole point of `fields`.

Each package also needs its `tsconfig.json` — this is the line people forget, and the error
is unhelpful:

```json
{ "extends": "@refrain/tsconfig/react-library.json", "include": ["src"] }
```

```bash
# Confirm all four got one
ls packages/*/tsconfig.json
```

---

## Step 5 — `apps/` (skeletons only)

Do not build these yet. Chapter 5 builds the web app and Chapter 7 builds the extension.

```bash
mkdir -p apps/web/src
mkdir -p apps/extension/entrypoints/sidepanel apps/extension/entrypoints/content
```

`apps/web/package.json`:

```json
{
  "name": "@refrain/web",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "vite",
    "build": "vite build",
    "typecheck": "tsc --noEmit",
    "preview": "vite preview"
  },
  "dependencies": {
    "react": "^19.0.0",
    "react-dom": "^19.0.0",
    "react-router": "^7.0.0",
    "@refrain/fields": "workspace:*",
    "@refrain/vault": "workspace:*",
    "@refrain/ui": "workspace:*",
    "@refrain/mapping": "workspace:*",
    "@refrain/extract": "workspace:*"
  },
  "devDependencies": {
    "vite": "^6.0.0"
  }
}
```

`apps/extension/package.json` — **note `wxt` is a `devDependency`:**

```json
{
  "name": "@refrain/extension",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "wxt",
    "build": "wxt build",
    "zip": "wxt zip",
    "postinstall": "wxt prepare",
    "typecheck": "tsc --noEmit"
  },
  "dependencies": {
    "@refrain/fields": "workspace:*",
    "@refrain/ui": "workspace:*",
    "@refrain/vault": "workspace:*",
    "@refrain/mapping": "workspace:*"
  },
  "devDependencies": {
    "wxt": "^0.20.0"
  }
}
```

> `"postinstall": "wxt prepare"` **is not optional.** It generates the `wxt/` types directory —
> `WxtAppConfig`, the entrypoint virtual modules, the auto-import declarations. **Without it
> `pnpm dev` fails with a type error on your first file**, and the error message does not mention
> `wxt prepare`. If you see "Cannot find module 'wxt/utils/define-background'" and you have never
> heard of `wxt prepare`, that is what is wrong.
>
> `wxt` **is a devDependency, not a dependency.** Nothing at runtime imports it. Putting a build
> tool in `dependencies` misrepresents what ships and drags its entire tree into production
> installs.

### The side panel is an entry point, not an app

```bash
# Chapter 7 builds these for real. Create the dirs now so the
# tree matches what you will be told to write.
apps/extension/entrypoints/
├── sidepanel/
│   ├── index.html
│   ├── main.tsx          # createRoot(StrictMode, <Panel/>)
│   └── App.tsx
└── content/
    └── index.ts          # the field scanner. No React, ever.
```

```ts
// apps/extension/wxt.config.ts
import { defineConfig } from "wxt"

export default defineConfig({
  srcDir: "entrypoints",
  modules: [],
  manifest: {
    name: "Refrain",
    permissions: ["sidePanel", "storage"],
    // Surgical, not <all_urls>. Chapter 18 explains why the
    // short list is a trust feature, not a limitation.
    host_permissions: ["https://forms.google.com/*"],
  },
  vite: () => ({
    build: { target: "chrome114" },   // sidePanel API landed in 114
  }),
})
```

> `build.target: "chrome114"` **is a deliberate floor, not a default you accepted.** The
> `sidePanel` API does not exist before 114, so a lower target produces a bundle that calls a
> function Chrome does not have — which fails silently until the user opens the panel and gets an
> empty white rectangle. Set the floor to your oldest supported version and **say so in the store
> listing**. "Chrome 114+" is a limitation a user can plan around; discovering it after install
> is a bad review.
>
> **WXT does not know about `sidePanel`.** It will not generate the `side_panel.default_path`
> entry for you because there is no `sidepanel.html` convention in its defaults the way
> `popup.html` is conventional. **Check the built `manifest.json` after your first build and
> confirm `side_panel` is present** — the panel not opening in the store version, with no error,
> is this bug.

### `pnpm-workspace.yaml` final check

Your glob is `apps/*` and `packages/*`. `pnpm-workspace.yaml` was written in Step 1. **Confirm
it is still there** — if you forgot it, nothing links and the error message is unhelpful.

---

## Step 6 — Install and boot

```bash
pnpm install
```

First run will be slow. pnpm will tell you about blocked build scripts — that is the
`onlyBuiltDependencies` list doing its job.

```bash
pnpm typecheck
```

Expected: **every package with a `typecheck` script reports success.** If anything fails, you
have a dependency cycle or a missing preset. Fix it now, not in Chapter 8 when a cycle makes an
unrelated test fail and you spend an hour blaming the wrong package.

> **Do not expect a count of seven.** `packages/tsconfig` has no scripts, so six workspaces
> report here: five `packages/*` plus `apps/web` and `apps/extension`. **Chapter 5 and 7 have
> not written the app entry points yet, so `apps/*` will report success trivially or error on a
> missing `vite.config.ts`.** Both are fine today. What you are proving is that the *library*
> packages typecheck.

```bash
pnpm dev
```

Expect **nothing to happen yet** — the apps have no entry points. That is correct. The goal
today is that the command *runs* and Turborepo *resolves the graph*. Add a throwaway
`console.log` to one `dev` script and confirm the wiring works, then delete it.

> **`pnpm dev` will hang, and that is success.** `dev` is `persistent: true`, so Turborepo
> starts it and waits forever. Ctrl-C to exit. If your command *returns* immediately, the
> `persistent` flag is missing and Turbo gave up — go back to `turbo.json`.

### Learn the filters — you will use these daily

```bash
pnpm --filter @refrain/extension dev     # one package
pnpm --filter "@refrain/ui..." dev       # ui AND everything it depends on
pnpm --filter "...@refrain/extension" dev # extension AND everything that depends on it
pnpm --filter "./apps/*" build            # everything under apps/
pnpm exec turbo run typecheck --dry=json  # show the task graph without running it
```

`--dry=json` **is the best debugging tool in this repo.** When Turborepo does something
unexpected, print the graph instead of guessing.

> **Read the dots before and after the package name.** `pkg...` means "this package and its
> dependencies." `...pkg` means "this package and its dependents." Getting these backwards is
> the most common `--filter` mistake, and the symptom is that nothing runs and you have no idea
> why. When in doubt, print the graph.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "chore(monorepo): pnpm + turborepo, 6 packages, 2 apps, shared tsconfig"
git commit --allow-empty -m "chore(monorepo): verified pnpm dev resolves the graph"
```

> **Check what you are about to commit.** This is the habit that protects §11:
>
> ```bash
> git status
> git diff --cached --stat
> ```
>
> If anything under `node_modules/` or a `*.pdf` appears, your `.gitignore` is wrong. Fix it
> before the commit, not after — rewriting history to remove a marksheet is far more work.

---

## Your 60/40 split for this chapter

**Every task in this chapter, in the order it appears. 24 tasks: 18 yours, 6 delegated.**

| # | Task | Who | Why |
| - | ---- | --- | --- |
| **Step 1 — the root** ||||
| 1 | `mkdir refrain`, `git init`, decide where it lives | **You** | Nesting the monorepo inside its own docs repo causes the `git status` mess in Chapter 4 |
| 2 | Root `package.json` incl. `packageManager`, `check` script | **You** | You will run `pnpm check` for 23 days |
| 3 | `pnpm-workspace.yaml` + `onlyBuiltDependencies` | **You** | ⚠️ Its absence means nothing links, and the error does not say so |
| 4 | `turbo.json` — `tasks`, `dependsOn`, `persistent` | **You** | You will debug this at 11pm, and you cannot debug a file you did not write |
| 5 | `.gitignore` incl. `*.pdf` `*.png` | **You** | ⚠️ **The §11 line.** A committed marksheet is a privacy violation with git as evidence |
| 6 | `tsconfig.base.json` | **You** | Shared rules for 8 packages. One edit, everything inherits |
| **Step 2 — presets** ||||
| 7 | `packages/tsconfig/{base,react-library,app}.json` | **You, once** | Then reuse forever. Twelve lines total |
| 8 | Understand why `packages/tsconfig` has no scripts | **You** | Otherwise you hunt for a missing `typecheck` report |
| **Step 3 — `packages/fields`** ||||
| 9 | `package.json` + `tsconfig.json` | **You** | You set `exports` yourself, and that choice is a real decision |
| 10 | **`src/index.ts` — every schema, field, and comment** | **You. Entirely.** | ⚠️ **The contract. See below** |
| 11 | The `.strict()` decision on `FormFieldSchema` only | **You** | It is a judgement call about silent vs loud failures |
| 12 | `withSuffix` and the iframe ID convention | **You** | Chapter 7 depends on this existing in one place |
| 13 | **The 7 tests, especially the fixed `.strict()` one** | **You** | ⚠️ The most valuable moment in the chapter |
| 14 | **Understand why the loose test passed for the wrong reason** | **You** | The single most transferable lesson in Chapter 2 |
| 15 | Run `pnpm test`, break a test, read the diff | **You** | Reading a failure is the skill |
| 16 | 8 more adversarial tests | **OpenCode** — then you audit each as below | Good 40% work: mechanical volume, high value if you check it |
| **Step 4 — the four libraries** ||||
| 17 | The dependency rule (who may import whom) | **You** | This is an architecture decision, not boilerplate |
| 18 | The `mkdir` loop | **You** | One line, and you see the shape |
| 19 | `package.json` × 4 for vault/mapping/extract/ui | **OpenCode** — paste the `fields` one as the template | Genuine boilerplate. Verify the `name` and `exports` changed |
| 20 | `tsconfig.json` × 4 | **OpenCode** | One identical line each |
| **Step 5 — the apps** ||||
| 21 | `apps/web/package.json` | **OpenCode** — verify the `workspace:*` list | Dependency list is mechanical |
| 22 | `apps/extension/package.json` + the `wxt` devDependency call | **You** | The dev-vs-dependency distinction is a concept, not boilerplate |
| 23 | `wxt.config.ts` — `chrome114` floor, `host_permissions`, the `side_panel` warning | **You** | ⚠️ Three real decisions with real failure modes |
| 24 | `entrypoints/` directory structure | **OpenCode** | Dirs and stub files. Chapters 5 and 7 replace them |

### The OpenCode prompts for this chapter

Paste these as-is. **Each one is followed by your audit step, which is not optional.**

**Task 16 — adversarial tests:**

```
Write 8 more adversarial Zod tests for @refrain/fields.
Focus on: partial objects, wrong enum values, missing required keys.
Do NOT change the source. Do NOT add new schemas.
For each test, add a one-line comment stating which schema rule it defends.
```

Then **your audit**, which is the actual 40%:

> For each test it wrote, ask: *what exactly would fail if I deleted this rule from the schema?*
> If you cannot answer in one sentence, the test is loose — rewrite it to assert on
> `error.issues[0]?.code` instead of just `success === false`.

**Task 19 — the four library packages:**

```
Create package.json for these 4 workspace packages, modelled on
@refrain/fields: vault, mapping, extract, ui.

For each: change `name` to @refrain/<name>, keep `exports` pointing at
./src/index.ts, and set dependencies to exactly:
  vault   → dexie, @refrain/fields
  mapping → @refrain/fields, zod
  extract → pdfjs-dist, tesseract.js, @refrain/fields
  ui      → react, @refrain/fields

Also create a tsconfig.json in each: {"extends":"@refrain/tsconfig/react-library.json","include":["src"]}
```

**Task 21 — the web app:**

```
Create apps/web/package.json for a React 19 + Vite app with scripts
dev/build/typecheck/preview, and dependencies react, react-dom,
react-router, and all five @refrain/* packages as workspace:*.
```

### Why the line is drawn exactly where it is

> **Everything above `packages/fields` is yours because you will debug it at 11pm and you
> cannot debug a file you did not understand.** Everything below it is delegation.
>
> **`packages/fields/src/index.ts` is 100% yours, and it is the only file in the chapter that
> is.** It is the contract that the web app, the extension, the mapping engine, the extractor,
> and eventually the API all import. **Chapter 16's entire argument for keeping the server in
> TypeScript rests on this one file** — if the schema lived in Python instead, that chapter
> would have no answer. **If you let OpenCode write it, you will not be able to explain why
> `FieldValue.value` is a string, and that is the one question that explains Chapter 7's worst
> bug.**

> **Your audit of task 16 is worth more than task 16 itself.** Six delegated tasks produce six
> files you did not write. Three of them — the tests — can be *wrong in a way that passes*, and
> the only defence is the question you ask about each. **Delegation moves the work; it does not
> move the responsibility.**

---

## Gotchas in this chapter

**Turborepo does nothing. Exit code 1, no tasks run.** Your `turbo.json` uses `pipeline`
instead of `tasks`. Turbo prints an explicit "Rename `pipeline` field to `tasks`" error. Fix the
key name.

**`pnpm typecheck` says `No tasks were run`.** No workspace has a `typecheck` script. Check
`packages/*/package.json` — the `scripts` block is missing or misspelled.

**`ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL` right after install.** A package's `build` script runs
before its upstream dependency's. That is `dependsOn: ["^build"]` missing from your task
definition.

**`Cannot find module '@refrain/ui'` in the extension.** Either you are missing the dependency
in `apps/extension/package.json`, or you have not re-run `pnpm install` since adding it.
pnpm only links what is declared — that is the strictness from Chapter 1.

**pnpm 10 blocks `esbuild` and Vite fails with a confusing binary error.** This is the
`onlyBuiltDependencies` block. Add the package name and re-run `pnpm install`.

**`tsconfig.base.json` edited but nothing changed.** Six files extend a preset that
overrides it. Change `packages/tsconfig/base.json` instead, then restart the TS server.

**Turborepo caches a broken build and keeps replaying it.** `pnpm exec turbo run build --force`.
Read that as "my source changed but Turborepo thinks nothing did."

**`pnpm dev` returns immediately instead of hanging.** `persistent: true` is missing from the
`dev` task.

**A test fails and you cannot tell which rule caused it.** Assert on
`r.error.issues[0]?.code`, not just on `success === false`. A test that passes for the wrong
reason is the failure mode described in Step 3.

**`npm run` works but `pnpm run` does not, or vice versa.** You have a `packageManager` field
pinning pnpm and are mixing package managers. Use `pnpm` everywhere, including in scripts.

---

## Verify before moving on

- [ ] `pnpm install` completes clean, no blocked-script warnings
- [ ] `pnpm exec turbo --version` reports 2.x
- [ ] `pnpm typecheck` passes in every package that has the script
- [ ] `pnpm dev` runs and Turborepo resolves the graph
- [ ] `@refrain/fields` tests pass — all seven, including the `.strict()` one
- [ ] You can explain `dependsOn: ["^build"]` and why `^` matters
- [ ] You can explain `tasks` vs `pipeline`, and that the failure is loud
- [ ] `pnpm exec turbo run typecheck --dry=json` prints a graph you can read
- [ ] You can explain what `...pkg` means versus `pkg...`
- [ ] You can explain why `FormSchema` is lenient and `FormFieldSchema` is strict
- [ ] `.gitignore` blocks `*.pdf` and `*.png`
- [ ] `git status` is clean and shows no `node_modules`
- [ ] Two commits in the log

---

## Check yourself before Chapter 3

1. **Why does `packages/fields` exist separately from `packages/mapping`?**
2. **A package you did not write imports `@refrain/vault` into `@refrain/ui`. What breaks and why?**
3. **Why does pnpm 10 require `onlyBuiltDependencies`?**
4. **You add a dependency to `packages/vault`. The web app still fails to build with "cannot find module". Why?**
5. **What is the difference between `apps/` and `packages/`? Give the test, not the label.**
6. **Why is `"exports": "./src/index.ts"` acceptable in a private monorepo?**
7. **Why is `FormFieldSchema` `.strict()` while `FormSchema` is not?**
8. **A test asserts `safeParse(bad).success === false` and you delete the field that made it
   bad. The test still passes. What was it actually verifying?**
9. **Why does `pnpm dev` hanging with no output mean it worked?**
10. **Three programs ship from two apps. Which two, and why is that one app?**

---

**Next: [Chapter 3 — The Design System](./03-design-system.md)** —
Tailwind v4 tokens, shadcn/ui, dark mode, `packages/ui`, the four mascot states, and the
provenance chips that become the visual signature of the product.