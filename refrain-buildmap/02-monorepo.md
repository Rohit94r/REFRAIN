# Chapter 2 — The Monorepo

> **Day 2 · Goal: `pnpm dev` boots all three surfaces at once.**
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

**`apps/` are things with an entry point and a build output. `packages/` are libraries.**
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

### `package.json` (root)

```json
{
  "name": "refrain",
  "version": "0.1.0",
  "private": true,
  "type": "module",
  "packageManager": "pnpm@10.32.1",
  "engines": { "node": ">=20" },
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

**`pnpm check` is the command you will run most.** It chains typecheck → lint → test → build
in the order where the cheapest, most-likely-to-fail check runs first. Put that in your
muscle memory now.

**Pin `packageManager`.** It tells Corepack which pnpm version your lockfile expects. Without
it, your teammate and CI can silently use different pnpm and produce different lockfiles.

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

### `turbo.json`

```json
{
  "$schema": "https://turbo.build/schema.json",
  "globalDependencies": ["**/.env.*local"],
  "globalEnv": ["NODE_ENV"],
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "outputs": ["dist/**", ".output/**", ".wxt/**"]
    },
    "typecheck": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "dev": {
      "cache": false,
      "persistent": true
    },
    "test": { "dependsOn": ["^build"], "outputs": [] },
    "lint": { "outputs": [] },
    "clean": { "cache": false }
  }
}
```

| Line | What it does, and why you need it |
|---|---|
| `"tasks"` | **Turborepo 2.x renamed `pipeline` → `tasks`.** Every tutorial you find shows `pipeline` and it silently does nothing. If your tasks are not running, this is 90% of why. |
| `"dependsOn": ["^build"]` | The `^` means *upstream dependencies first*. `mapping` builds before `sidepanel`. Without it you get "cannot find module" half the time. |
| `"persistent": true` | Marks `dev` as a long-running process. Without it Turborepo waits for it to exit and hangs forever. |
| `"cache": false` on dev | Caching a watch process is nonsense. |
| `"outputs"` | What Turborepo is allowed to cache. If you omit it, builds are cached but the output is not restored and nothing works. |

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

---

## Step 3 — `packages/fields` (start here — everything depends on it)

The canonical definition of "a field." **Zero dependencies.** This is the contract between
your UI and your mapping engine.

```bash
mkdir -p packages/fields/src && cd packages/fields
pnpm init
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

> **Note `"exports": { ".": "./src/index.ts" }` — you are exporting source, not a build
> artifact.** In a private monorepo this is simpler and faster than compiling every package.
> Vite and Turborepo both handle it. If you ever need a real build, you flip this and add
> `dist/` to `exports`. Do not build the whole toolchain twice.

`src/index.ts`:

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

export const FormFieldSchema = z.object({
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

### Why these seven decisions matter

| Decision | What it prevents |
|---|---|
| `value` is always `string` | Chapter 5's silent-empty-submit bug. This is the single most important line in the file. |
| `confidence` requires a `reason` | You will never be able to show "why" on the review screen. There is no bare-float fallback. |
| `provenance` is required, not optional | Optional means skipped. Skipped means the review screen lies. |
| `editedByUser` | This is your **Phase 1 success metric** — correction rate must *decrease*. You cannot measure it without this flag. |
| `REVIEW_THRESHOLD` is one constant | One place to change the bar. Never a magic number in a resolver. |
| `blocked` on the form | §11's hard "no" list is enforced in the **schema**, so it is impossible to forget at the call site. |
| `frameId` | Chapter 5's iframe gotcha, solved in the type instead of in a bug report. |

### `src/index.test.ts` — write this yourself

```ts
import { describe, it, expect } from "vitest"
import { FormSchema, REVIEW_THRESHOLD } from "./index"

describe("FormSchema", () => {
  it("rejects a field whose value is not a string", () => {
    const bad = { url: "u", title: "t", fields: [{ value: 87.4 }] }
    expect(FormSchema.safeParse(bad).success).toBe(false)
  })

  it("requires provenance on every field", () => {
    const noProvenance = {
      url: "u", title: "t",
      fields: [{ id: "1", label: "Name", kind: "text", required: false }],
    }
    expect(FormSchema.safeParse(noProvenance).success).toBe(false)
  })
})
```

**Run it: `pnpm test`.** Then deliberately break the test and read the failure.

> **This is the moment to start using OpenCode.** Ask it:
> *"Write 8 more adversarial Zod tests for `@refrain/fields` — focus on partial objects,
> wrong enum values, and missing required keys. Do not change the source."*
> Reading its output and understanding why each test exists **is** the 40%.

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

| Package | Depends on | Why |
|---|---|---|
| `vault` | `dexie`, `@refrain/fields` | It stores field-typed documents |
| `mapping` | `@refrain/fields`, `zod` | Pure logic, no UI, no DOM |
| `extract` | `pdfjs-dist`, `tesseract.js`, `@refrain/fields` | Document → field values |
| `ui` | `react`, `@refrain/fields` | Components only. **Never depends on `mapping` or `vault`** |

> **The dependency rule, memorise it:** `fields` ← everyone. `ui` depends on `fields` only.
> `mapping` depends on `fields` only. `vault` depends on `fields` only. The apps wire them
> together. **Packages must not import each other sideways.** If `vault` needs `mapping`, you
> have put business logic in the wrong box.

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

> **`"postinstall": "wxt prepare"` is not optional.** It generates the `wxt/` types directory —
> `WxtAppConfig`, the entrypoint virtual modules, the auto-import declarations. **Without it
> `pnpm dev` fails with a type error on your first file**, and the error message does not mention
> `wxt prepare`. If you see "Cannot find module 'wxt/utils/define-background'" and you have never
> heard of `wxt prepare`, that is what is wrong.
>
> **`wxt` is a devDependency, not a dependency.** Nothing at runtime imports it. Putting a build
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
    build: { target: "chrome114" },   // the sidePanel API landed in 114
  }),
})
```

> **`build.target: "chrome114"` is a deliberate floor, not a default you accepted.** The
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

Expected: **every package reports success.** If anything fails, you have a dependency cycle or
a missing preset. Fix it now, not in Chapter 8 when a cycle makes an unrelated test fail and you
spend an hour blaming the wrong package.

```bash
pnpm dev
```

Expect **nothing to happen yet** — the apps have no entry points. That is correct. The goal
today is that the command *runs* and Turborepo *resolves the graph*. Add a throwaway
`console.log` to one `dev` script and confirm the wiring works, then delete it.

### Learn the filters — you will use these daily

```bash
pnpm --filter @refrain/extension dev     # one package
pnpm --filter "@refrain/ui..." dev       # ui AND everything it depends on
pnpm --filter "...@refrain/extension" dev # extension AND everything that depends on it
pnpm --filter "./apps/*" build            # everything under apps/
pnpm turbo run typecheck --dry=json       # show the task graph without running it
```

**`--dry=json` is the best debugging tool in this repo.** When Turborepo does something
unexpected, print the graph instead of guessing.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "chore(monorepo): pnpm + turborepo, 6 packages, 2 apps, shared tsconfig"
git commit --allow-empty -m "chore(monorepo): verified pnpm dev resolves the graph"
```

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| `pnpm-workspace.yaml`, root `package.json`, `turbo.json` | **You** — this is the part you will debug |
| `packages/tsconfig/*` presets | **You**, once. Then reuse forever |
| **`packages/fields/src/index.ts` — every Zod schema and comment** | **You. Entirely.** This is the contract |
| Adversarial tests for `fields` | **OpenCode** — then you read and explain each one |
| Boilerplate `package.json` for vault/mapping/extract/ui | **OpenCode** — paste the `fields` one as the template |
| `apps/*` skeletons | **OpenCode** — they are placeholders until Chapters 5 and 7 |

---

## Gotchas in this chapter

**Turborepo does nothing. No errors, no output.** Your `turbo.json` uses `pipeline` instead of
`tasks`. You followed a v1 tutorial. Fix the key name.

**`ERR_PNPM_RECURSIVE_RUN_FIRST_FAIL` right after install.** A package's `build` script runs
before its upstream dependency's. That is `dependsOn: ["^build"]` missing from your task
definition.

**`Cannot find module '@refrain/ui'` in the extension.** Either you are missing the dependency
in `apps/extension/package.json`, or you have not re-run `pnpm install` since adding it.
pnpm only links what is declared — that is the strictness from Chapter 1.

**pnpm 10 blocks `esbuild` and Vite fails with a confusing binary error.** This is the
`onlyBuiltDependencies` block. Add the package name and re-run `pnpm install`.

**`tsconfig.base.json` edited but nothing changed.** Seven files extend a preset that
overrides it. Change `packages/tsconfig/base.json` instead, then restart the TS server.

**Turborepo caches a broken build and keeps replaying it.** `pnpm turbo run build --force`.
Read that as "my source changed but Turborepo thinks nothing did."

---

## Verify before moving on

- [ ] `pnpm install` completes clean, no blocked-script warnings
- [ ] `pnpm typecheck` passes in all 9 packages
- [ ] `pnpm dev` runs and Turborepo resolves the graph
- [ ] `@refrain/fields` tests pass, including the two adversarial ones
- [ ] You can explain `dependsOn: ["^build"]` and why `^` matters
- [ ] You can explain `tasks` vs `pipeline`
- [ ] `pnpm --filter ... --dry=json` prints a graph you can read
- [ ] `.gitignore` blocks `*.pdf` and `*.png`
- [ ] Two commits in the log

---

## Check yourself before Chapter 3

1. **Why does `packages/fields` exist separately from `packages/mapping`?**
2. **A package you did not write imports `@refrain/vault` into `@refrain/ui`. What breaks and why?**
3. **Why does pnpm 10 require `onlyBuiltDependencies`?**
4. **You add a dependency to `packages/vault`. The web app still fails to build with "cannot find module". Why?**
5. **What is the difference between `apps/` and `packages/`? Give the test, not the label.**
6. **Why is `"exports": "./src/index.ts"` acceptable in a private monorepo?**

---

**Next: [Chapter 3 — The Design System](./03-design-system.md)** —
Tailwind v4 tokens, shadcn/ui, dark mode, `packages/ui`, the four mascot states, and the
provenance chips that become the visual signature of the product.
