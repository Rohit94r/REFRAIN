# Tech Stack — Refrain

> Every technology in Refrain, why it is here, what version, and what you would swap it for.
>
> This is the reference. Chapters teach the *how*; this file is the *what* and the *why not*.

---

## The one-paragraph version

A **TypeScript monorepo** (pnpm + Turborepo) containing a **React 19** front end, a **WXT**
Chrome MV3 extension, four shared libraries, and a **Hono + MongoDB** backend that exists only
for optional sync. All data is **encrypted on the device with AES-256-GCM** before it leaves.
Inference is **on-device** — Chrome's `LanguageModel` API, falling back to Ollama. There is no
cloud model provider anywhere in the tree, and there never will be.

---

## Part A — Runtime & Toolchain


| Tech           | Version       | Why this one                                                                                                                                                                                                                   | Swap for                                                 |
| -------------- | ------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ | -------------------------------------------------------- |
| **Node.js**    | 22 LTS or 26  | Not "Node is better" — **MV3 is JavaScript-only, so two of three surfaces had no choice.** Node's `crypto.subtle` also matches the vault's WebCrypto calls, so Chapter 6's crypto is portable rather than rewritten. See below | Bun (works, but Turborepo + pnpm is a solved path)       |
| **pnpm**       | 10.x          | Content-addressable store. A package **cannot import what it did not declare** — that strictness is what keeps a 7-package monorepo sane                                                                                       | npm (flat node_modules, haunted-house dependency graph)  |
| **Turborepo**  | 2.x           | Task graph with `dependsOn: ["^build"]`. Caching is free                                                                                                                                                                       | Nx (heavier), Nx/none (you re-do build ordering by hand) |
| **TypeScript** | 5.9+          | Your product's core value is *provenance* — knowing where every value came from. That is a data-shape problem and TypeScript is the tool for data-shape problems                                                               | Flow, JSDoc (both worse)                                 |
| **ESLint**     | 9 flat config | Flat config, no `.eslintrc` inheritance puzzles                                                                                                                                                                                | Biome (faster, smaller rule set)                         |
| **Prettier**   | 3.x           | Formatting you do not have to think about                                                                                                                                                                                      | Biome (`--write` does both)                              |
| **Vitest**     | 3.x           | Native ESM + TS. Same runner in every package. `watch` is free                                                                                                                                                                 | Jest (slow, ESM pain), node:test (no watch, no mocks)    |


> **Why pnpm and not npm, restated because it is the decision you will defend most.**
> pnpm only links what is declared in a package's own `package.json`. In a monorepo where
> `ui`, `mapping`, `vault` and `fields` all exist, that strictness is the entire reason your
> dependency graph stays legible. It will feel like a bug in Chapter 3. It is not.

### Why Node, and not Python or a Next.js backend

This gets asked within a day of starting, so the answer is written down rather than defended
later. Read the whole section before accepting the choice — the reasoning is not "Node is
better," it is "Node is the only option that keeps one contract."

**Two of the three surfaces have no choice at all.**


| Surface          | Runtime                  | Was this a decision?                                                 |
| ---------------- | ------------------------ | -------------------------------------------------------------------- |
| Chrome extension | V8, JavaScript only      | ❌ **No.** MV3 has no other runtime. There is no extension in Python. |
| Web app          | Browser, JavaScript only | ❌ **No.** Same.                                                      |
| Sync API         | *Your choice*            | ✅ This is the only real question in the room                         |


So the question is not "should the whole project be Python." It is **"does the one server
deserve its own language?"** And that turns on a single package.

### The decisive reason: `packages/fields` is imported by both sides

```
              packages/fields/
              FormField · ProfileFact · SyncEntry · ErrorCode
                        ▲                    ▲
                        │                    │
             apps/extension            apps/api
          (the thing that ships)    (the optional part)
```

One file. One schema. Both sides of the wire. Chapter 16's `zValidator("json", PushRequest)`
imports the *same* `SyncEntry` type the extension builds its payload from.

> **This is the whole answer.** A validation shape change fails CI, not production, because
> there is only one shape to change. Everything else below is secondary.

Now trace what happens in Python. You write `class SyncEntry(BaseModel)` in FastAPI, and
TypeScript already has `SyncEntrySchema` in Zod. **Two definitions of the sync protocol.** And
the failure is silent, because nothing compares them.

> **The concrete drift bug.** Chapter 15 adds `sourceDeviceId` to `SyncEntry`. In TypeScript you
> add it to the Zod schema once, the API's `zValidator` starts rejecting every client that omits
> it, and you find out in CI on a Tuesday. In Python you add it to the Pydantic model, the
> extension does not know about it, and **nothing fails anywhere.** The sync silently writes a
> `undefined` that MongoDB coerces to `null`, which your three-way merge then reads as a
> deliberate value and happily overwrites your other device's edit with it. That is Chapter 15's
> hardest bug, and the contract-sharing is what prevents it.

### Python — the honest accounting


| For                                       | Against                                                                                                                                                                                                                                                                                      |
| ----------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **The best ML ecosystem in existence**    | **Irrelevant.** Inference is on-device in Chrome's `LanguageModel` or Ollama. Both are HTTP/JSON endpoints you call *from TypeScript*. The server never runs a model.                                                                                                                        |
| `pytesseract` + OpenCV for OCR            | **Irrelevant, and inverted.** OCR happens **in the browser** with `tesseract.js` because the marksheet must never be uploaded. Chapter 11. The server never sees the image.                                                                                                                  |
| FastAPI + Pydantic is genuinely excellent | **You now maintain two schema languages.** Above.                                                                                                                                                                                                                                            |
|                                           | **Your crypto exists already.** Chapter 6 wrote PBKDF2, AES-GCM, and HKDF against WebCrypto's `deriveBits` / `importKey` / `subtle.deriveKey`. Rewriting that in a second language, for the same algorithms with different API shapes, is a security risk taken on for zero functional gain. |
|                                           | `**subtle.sign` for EdDSA is JS.** Chapter 14's keypair. Node keeps it in one file.                                                                                                                                                                                                          |
|                                           | **Two of three surfaces stay TypeScript anyway.** You would end up with TS + Python, not one language.                                                                                                                                                                                       |
|                                           | **Two toolchains, two linters, two test runners, two lockfiles** in a repo designed to have one of each.                                                                                                                                                                                     |


> **The strongest argument for Python here is also the argument against it.** If your server
> needed to run inference, Python would win outright. Your server cannot — that is the entire
> product. **Choosing Python for ML you are forbidden from doing is the clearest signal that
> the language choice has drifted away from the requirements.**

### Next.js as the backend — the most tempting wrong answer

This one gets proposed most often, and it sounds maximally efficient: one framework, one
language, one deploy. Six reasons it does not work here, and the first is fatal on its own.

**1. Your extension cannot call a Server Action. Ever.**
Server Actions are bound to the React Server request runtime. Your sync client is the content
script and the side panel — a `chrome.runtime.sendMessage` call and a `fetch`. It is not a React
component in a Next request, and it never will be. So you write API routes for the extension
anyway, and then you have **two request paths**: API routes for the extension (which is the
entire point of the backend) and Server Actions for the web app (which needs almost nothing from
it). The duplication is inverted from what you would want, and you pay for two.

**2. Cold starts land on the exact path you optimised for.**
Sync fires on `visibilitychange`, on `online`, and after every fill. A cold Next function is
500ms–1s *before your code runs*. Chapter 18 budgets panel interactive at 400ms and treats
latency as a product-quality signal, not an infrastructure detail. A cold lambda in front of a
sync queue is a spinner.

**3. Blobs do not want serverless.**
A 40MB ciphertext push hits lambda request-size limits and wall-clock timeouts. You want a
long-lived container that streams to disk. That is Fly.io with Hono — which is exactly what
Chapter 17 deploys.

**4. It couples API uptime to web deploys.**
Chapter 17's entire rollback story assumes three independently deployable surfaces: roll the API
back without touching the web app, in 90 seconds. One monolith removes that option, and couples
your sync availability to a front-end release.

**5. Local development gets worse, not better.**
Chapter 12's promise is *one command, no Docker required.* A plain Hono server is `node --watch`.
Next gives you a dev server, and if you want the API deployable independently you are back to a
second process anyway.

**6. You lose the hosting choice.**
Hono runs on Node, Fly, Render, **and Cloudflare Workers** from the same source. Next runs on
Node or Vercel. Chapter 17's hosting table says *"pick based on where your MongoDB region is."*
A monolith collapses that to "Vercel, and Vercel is in whichever region Vercel is in."

### Where Node loses, stated plainly


| Cost                                                             | How bad is it here?                                                                                                                                                                                                                                                   |
| ---------------------------------------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Argon2id needs a native module** (`argon2`, `@node-rs/argon2`) | **The real tax.** A build dependency: CI needs the toolchain, the Alpine container needs `build-base` or a prebuilt binary, and native modules are the single most common source of "works on my machine." Chapter 12's Docker build is the place you will feel this. |
| **CPU-bound work blocks the event loop**                         | **Not currently a problem.** Sync is I/O plus small crypto. But one slow handler blocks every concurrent request, and there is no preemption.                                                                                                                         |
| **Single-threaded, no parallelism for free**                     | **You would reach for a worker thread or a queue** — both of which are exactly the complexity Chapter 18's "did not build" table refuses to add.                                                                                                                      |
| Deep recursion / GIL-free numerics                               | Not relevant.                                                                                                                                                                                                                                                         |


> `**scrypt` from `node:crypto` is the escape hatch.** Chapter 12 lists Argon2id as
> "acceptable: `scrypt` is built in." If the native module fights you in CI or in the Alpine
> image, **switch to `scrypt` and record it as a §19 decision.** A native build that fails at
> 1am on deploy day is a worse outcome than `scrypt` with a documented cost factor. Do not let
> a dependency win an argument about dependencies.

### When the answer flips


| If this became true                                  | Then choose                                                                                                                                     |
| ---------------------------------------------------- | ----------------------------------------------------------------------------------------------------------------------------------------------- |
| OCR or image processing moves **server-side**        | **Python or Rust.** But that requires uploading the marksheet, which breaks §11. **The constraint refuses this option before you can take it.** |
| You needed server-side embeddings or semantic search | **Python.** But the server cannot read the data, so it cannot embed it. Self-refusing again.                                                    |
| You were building *only* the API, with no extension  | **Python, probably.** Nothing would constrain you.                                                                                              |
| You needed to stream 100MB+ blobs                    | **Go or Rust.** Not Next, and not more Node tuning.                                                                                             |
| Nothing changed                                      | **Node.** The shared contract is worth more than any language's ergonomics.                                                                     |


> **Notice the shape of that table.** Every row where Python wins requires you to **break the
> end-to-end encryption property or move inference server-side — both of which break the
> product's central promise.** The constraint that chose Node also permanently forbids the
> language that would suit it. **That is not a coincidence; it is the same fact seen twice.**
> Local-first is not a feature you chose despite the stack. It chose the stack.

---

## Part B — Front End


| Tech                      | Version | Why                                                                                                                  | Notes                                                        |
| ------------------------- | ------- | -------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------ |
| **React**                 | 19      | Your side panel and web app are *the same app with two doors*. One component tree, two entry points                  | —                                                            |
| **Vite**                  | 7.x     | Dev server with HMR; ESM-native; fast builds                                                                         | —                                                            |
| **WXT**                   | 0.20+   | Extension framework built for multiple entry points (side panel + content script) sharing one React codebase         | Plasmo, `@crxjs/vite-plugin`                                 |
| **Tailwind CSS**          | **v4**  | CSS-first config in `@theme`. No `tailwind.config.js`.                                                               | **v3 tutorials are wrong for you**                           |
| **shadcn/ui**             | latest  | Copies source into your repo, so you own `Button` and can make it do what Fermata needs                              | Mantine, Radix primitives (Radix is what shadcn wraps)       |
| **Zustand**               | 5.x     | 300 bytes. One store for panel state. No context nesting                                                             | Redux Toolkit (heavier), Jotai (atom-per-value gets awkward) |
| **XState**                | v5      | Fermata's state machine has **guards** (§11 refusals, CAPTCHA). Guards belong in the machine, not in `if` statements | Reducer + switch (fine until you need guards)                |
| **React Hook Form + Zod** | latest  | Inline editing in a 40-field review list                                                                             | Controlled inputs (fine at 4 fields, painful at 40)          |
| **@tanstack/react-query** | 5.x     | Server state for sync status                                                                                         | —                                                            |
| **Dexie**                 | 4.x     | IndexedDB with real indexes, transactions, and `useLiveQuery`. IndexedDB without Dexie is unusable                   | Raw IndexedDB (untyped, no indexes in JS)                    |
| **Recharts**              | 3.x     | The Setlist's correction-rate trend. That is the only chart in Phase 1                                               | —                                                            |
| **oklch()**               | CSS     | Perceptual colour. One lightness → a whole ramp that reads consistently                                              | Hex (no perceptual lightness to hold)                        |


> **WXT over Plasmo is a deliberate deviation from `refrain.md` §10.** §10 lists Plasmo and
> `@crxjs/vite-plugin`. WXT models entry points as *files*, so the side panel entry imports
> `@refrain/ui` and `@refrain/vault` exactly like the web app does — no shared-build
> gymnastics. Plasmo is opinionated about its own bundler and fights pnpm workspaces.
> Record this in §19. Chapter 3 has the full comparison.

---

## Part C — The Vault & Data


| Tech                                       | Why                                                                                                                                                                            |
| ------------------------------------------ | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **IndexedDB**                              | The only storage that holds more than ~5MB and works offline in an extension context. `localStorage` is synchronous and 5MB — it will corrupt your profile and stall the panel |
| **AES-256-GCM** (WebCrypto)                | Authenticated encryption. Tamper-evidence is free — do not write your own                                                                                                      |
| **PBKDF2-SHA256, 600k iterations** (OWASP) | CPU-hard key derivation, native in WebCrypto, no WASM. ~250ms, which *feels considered*                                                                                        |
| **Argon2id**                               | Memory-hard, the correct answer. Needs WASM. **Planned upgrade**, gated on a versioned KDF id so old vaults re-encrypt losslessly                                              |
| **No key in `localStorage`**               | The whole point. `extractable: false` makes it structurally impossible                                                                                                         |
| **Fresh IV per record**                    | Reusing an IV under GCM leaks the auth key. `crypto.getRandomValues` inside `seal()` makes reuse impossible by construction                                                    |


> **Why not `localStorage` + a library like `crypto-js`.** `crypto-js` is unmaintained, slow,
> and requires you to manage IVs, salts, and key derivation yourself — which is how people
> ship broken crypto. WebCrypto is in the browser, audited by browser vendors, and gets
> faster every release. There is no good argument for a crypto library here.

---

## Part D — Extension & Content Script


| Tech                              | Why                                                                                                                                                                                                                               |
| --------------------------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Chrome MV3**                    | The only manifest version the Web Store accepts                                                                                                                                                                                   |
| `**sidePanel` API**               | Chrome 114+. Turns the side panel into essentially a web page — which is *why* `packages/ui` can be shared                                                                                                                        |
| `**allFrames: true**`             | Without it you report zero fields on any iframe portal. This single flag is the difference between "works" and "doesn't"                                                                                                          |
| **Native property setter**        | React monkey-patches `value` on the element instance. Writing `el.value = x` updates the DOM and React never hears about it — so the form submits empty while the user *sees* the text. The most important 8 lines in the product |
| `**composed: true**` on events    | Without it events do not escape a shadow root and the framework never sees them                                                                                                                                                   |
| **Hidden-input driver detection** | Modern widgets show a styled div and hold the real value in a hidden input. Fill the decoy and nothing is stored                                                                                                                  |


---

## Part E — Documents & Extraction


| Tech                        | Why                                                                                   |
| --------------------------- | ------------------------------------------------------------------------------------- |
| **pdfjs-dist**              | The only mature in-browser PDF text-layer reader. Free, no WASM server, works offline |
| **Tesseract.js**            | OCR for photographed marksheets — most Indian marksheets have no text layer           |
| **Canvas 2× rasterisation** | Tesseract needs pixels, not PDF bytes. Under 2× OCR on a phone photo is unusable      |
| **pdf.js worker**           | Mandatory. `workerSrc` unset throws immediately                                       |


> **The OCR confidence rule is a safety decision, not a tuning choice.** pdf.js text-layer
> extraction scores **0.95** (above `REVIEW_THRESHOLD`, fills). Tesseract OCR scores
> **0.62** (below it, never auto-fills). A name read from a photographed marksheet might be
> `Rohit Jadhav`, `Rohlth Jadhav`, or `Rohit | Jadhav`. At 0.62 it waits for one human
> confirmation — and then it is `manual` provenance at 1.0 forever, and every future form
> fills correctly because of that one tap. **That is what makes OCR safe to ship at all.**

---

## Part F — Inference (Local Only)


| Tier         | Tech                                            | Notes                                                                                                                            |
| ------------ | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------- |
| **Primary**  | Chrome `LanguageModel` API (Gemini Nano)        | Chrome 138+. Free, private, already on the machine. **The browser manages the model download and caches it outside your bundle** |
| **Fallback** | Ollama at `localhost:11434`                     | For users below Chrome 138 or who prefer a local model they control                                                              |
| **Never**    | OpenAI / Anthropic / Gemini API                 | §11's local-first claim is the moat. One `fetch` to a model provider spends it                                                   |
| **Never**    | `transformers.js`, `langchain`, bundled weights | A shipped model is 500MB+. Turns a 3MB extension into a 500MB extension                                                          |


> **Why the model is last in the pipeline, not first.** Rules resolve ~92% of fields, cost
> ₹0, run in sub-millisecond, and produce a *reason string* you can print on a chip. The
> model handles the ≤8% leftovers. `temperature: 0` matters more here than anywhere else in
> the product — mapping is not creative writing, and an unstable mapping makes the review
> screen untrustworthy.

---

## Part G — Backend (Optional Sync Only)


| Tech                                    | Version                 | Why this one                                                                                                                                                             |
| --------------------------------------- | ----------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| **Hono**                                | 4.x                     | Tiny, Web-Standard `Request`/`Response`, runs on Cloudflare Workers *or* Node. If you ever move off Atlas you can deploy the same code to Workers                        |
| **MongoDB + Atlas**                     | 8.x                     | Your data is a **document graph** with open-ended attributes. MongoDB models that natively; a relational schema would need a `jsonb` column and you would be fighting it |
| **Mongoose**                            | 8.x                     | Schema validation, indexes, middleware, populate. Real discipline on top of a schemaless store                                                                           |
| **Argon2id**                            | —                       | Password hashing. Node's built-in `scrypt` is acceptable; Argon2id is better                                                                                             |
| **Zod**                                 | 4.x                     | Shared with the frontend. **One request schema, both sides** — a shape change fails CI, not production                                                                   |
| **HKDF-SHA256**                         | WebCrypto / node:crypto | Splits one passphrase into `k_auth` (server) and `k_vault` (never sent). See below                                                                                       |
| **JWT** (access) + opaque refresh token | —                       | Access token 15 min in memory. Refresh token httpOnly, rotated on every use                                                                                              |


### The three hard truths about MongoDB + E2EE

You asked for MongoDB for the long run. It is the right call for the data shape. It creates
three real tensions, and pretending otherwise is how you end up shipping something you cannot
defend:

**1. The "no server" claim has to change.** Local-first stays the default and sync is opt-in,
but the honest line becomes: *"Works entirely offline. Sync is optional, and encrypted with a
key the server never sees."* You must update §11, the privacy policy, and the Web Store
disclosure. Chapter 12 does this explicitly.

**2. A server that cannot read your data cannot query your data.** If you store opaque
ciphertext, you have thrown away every index, aggregate, and server-side search MongoDB exists
for. That is the actual cost of end-to-end encryption, and it is not small.

**3. So: hybrid storage.** Two collections.


| Collection | Contents                                                              | Server can read it? |
| ---------- | --------------------------------------------------------------------- | ------------------- |
| `blobs`    | Opaque encrypted payloads + IV + wrapped DEK                          | ❌ No                |
| `syncmeta` | `userId`, `docId`, `updatedAt`, `deviceId`, `tombstone`, `byteLength` | ✅ Yes               |


`syncmeta` carries **no personal content.** It exists so sync can resolve conflicts, propagate
deletions, and show "this document changed on your phone." That is the minimum and it is
genuinely necessary — without tombstones, deletions never propagate and deleted documents
resurrect forever. Chapter 15 builds it.

> **Corollary: search must be client-side.** Atlas Search cannot index ciphertext. Your
> profile search runs in the browser over the decrypted in-memory store. That is a real
> constraint on a large vault, and Chapter 18 handles it with a local inverted index rather
> than pretending it does not exist.

---

## Part H — Infrastructure


| Concern                | Choice                                        | Why                                                                                                                                   |
| ---------------------- | --------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------- |
| **Hosting**            | Fly.io or Render (Node) or Cloudflare Workers | Hono runs on all three. Pick based on where your MongoDB region is, to minimise latency on sync                                       |
| **Database**           | MongoDB Atlas M10+                            | Managed backups, point-in-time restore, replica set. Do not self-host MongoDB on day one                                              |
| **Connection pooling** | `mongodb-connection-pooler` / Atlas pooler    | Serverless-style scaling without exhausting the 10,000-connection-per-cluster limit                                                   |
| **CI**                 | GitHub Actions                                | Matrix Node versions, `pnpm check`, build artefact                                                                                    |
| **CD**                 | GitHub Actions → Fly/Render                   | Extension is built as a signed `.zip` artefact — the Web Store takes a zip, not a deploy                                              |
| **Monitoring**         | Sentry (server only)                          | **Never in the extension.** Server errors only, scrubbed. Adding Sentry to the extension contradicts §11 and the Web Store disclosure |
| **Secrets**            | Host env vars + Atlas IP allowlist            | Never a `.env` committed, never in the bundle                                                                                         |
| **Rate limiting**      | Redis, or Atlas's built-in                    | Per-IP on `/auth/`*, per-user on `/sync`                                                                                              |
| **Backups**            | Atlas PITR                                    | Encrypted blobs are worthless without a restore plan                                                                                  |


> **Sentry in the server is defensible. Sentry in the extension is not.** The Web Store
> privacy practices form asks whether you collect usage data. If your monitoring SDK can see
> which forms a user opened, you have quietly converted a local-first product into a tracking
> product, and your §11 claim is false. Server errors only, never content, never field labels.

---

## Part I — Things deliberately absent


| Absent                                | Why it stays absent                                                                               |
| ------------------------------------- | ------------------------------------------------------------------------------------------------- |
| Any cloud LLM API                     | The moat is the arithmetic of ₹0 COGS                                                             |
| Telemetry / analytics SDKs            | §11. The moat                                                                                     |
| CAPTCHA solving services              | §11 no-list                                                                                       |
| Selenium / Playwright in shipped code | Bulk submission is the thing this product refuses to be                                           |
| Redis **required**                    | Nice for rate limiting, not a hard dependency. Do not add a datastore to avoid adding a datastore |
| A CSS framework                       | Tailwind + your tokens. `shadcn` copies source, so it is not a runtime dependency                 |
| Redux, Jest, Moment.js                | None of these earn their weight here                                                              |


---

## Version pinning — the thing that saves you

```jsonc
// package.json (root)
{
  "packageManager": "pnpm@10.32.1",   // ← Corepack honours this. Lock it.
  "engines": { "node": ">=22" }
}
```

Commit `pnpm-lock.yaml`. **Never use `--no-lockfile` and never run `pnpm update` casually** —
in a 7-package monorepo a stray bump moves five transitive versions at once and the error
message will point somewhere else entirely.

```bash
# What broke, and when?
pnpm why zod
pnpm outdated --recursive
```

> `**pnpm why` is the single most useful command in a monorepo.** When a type error appears in
> a package you did not touch, this tells you which dependency introduced it in seconds.

---

## Quick reference — the command you actually run

```bash
pnpm check          # typecheck → lint → test → build. The one habit.
pnpm dev            # turbo dev, all three surfaces
pnpm --filter @refrain/extension dev
pnpm --filter "@refrain/ui..." dev          # ui + its deps
pnpm --filter "...@refrain/extension" dev   # sidepanel + its dependents
pnpm turbo run build --force                # Turborepo cached something stale
pnpm turbo run typecheck --dry=json         # print the task graph
pnpm why <pkg>                              # who pulled this in
```

---

*Every technology here has a reason and a swap. If you cannot state the reason, delete the
dependency — a 3MB extension that does one job well is worth more than a 12MB one that does
fifteen.*