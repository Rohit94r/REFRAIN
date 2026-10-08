# Refrain — Build Map

> **Answer once. Fill everywhere.**
>
> **20 chapters. Day 1 to production. Backend first, then the frontend —
> the order a real company builds in, because the API is the contract and the
> UI is a consumer of it.**

`refrain.md` is the product spec. **This folder is the curriculum** — the order
to build in, the concept behind each step, the exact commands, and the reasoning
for every architectural decision.

---

## How to use this — your daily routine

**This is the workflow. Do it the same way every single day.**

```
┌──────────────────────────────────────────────────────────────┐
│ 1. READ the whole chapter first                              │
│    Including "Words you need to know" and "Understand this   │
│    first". Do not skip to the code.                          │
│                                                              │
│ 2. SAY the answers out loud                                 │
│    The 10 questions at the end. BEFORE looking at them.     │
│    If you cannot answer, reread that part.                  │
│                                                              │
│ 3. BUILD it — "OpenCode, complete chapter N"                 │
│    Give it the chapter number. It reads the same chapter.    │
│    It writes the code. You watch what it writes.             │
│                                                              │
│ 4. CHECK everything it wrote                                │
│    Run `pnpm check`. Open the app. Click the buttons.        │
│    Do not trust it. Verify it.                              │
│                                                              │
│ 5. FIX IT YOURSELF by hand                                  │
│    Anything wrong, unclear, or ugly — you change it          │
│    yourself. This is the part that teaches you.             │
│                                                              │
│ 6. PUSH                                                       │
│    Commit your changes with a message that explains WHY.     │
└──────────────────────────────────────────────────────────────┘
```

### The rule that makes step 5 work

> **If OpenCode wrote a line and you cannot explain it, change it until you
> can.** You will never be asked about the parts you delegated. You will
> always be asked about the parts you understood.

Do not feel bad rewriting everything it did. That is the point. Step 5 is the
only step that makes this a learning project instead of a code-generating
project.

### A prompt that works

```
Complete chapter N from refrain-buildmap. Read the chapter first.
Explain each file you create and what it does. Comment the code so I can
read it later. Do not add anything the chapter does not ask for.
After writing, tell me what to run and what output I should see.
```

That last sentence matters — it makes it tell you how to check its own work.

---

## Chapter structure — what is inside every chapter

| Section | What it is for you |
|---|---|
| **Words you need to know** | Jargon explained in plain words before you meet it |
| **Understand this first** | The idea behind the step. No code. Read this or the code means nothing |
| **Step 1, 2, 3…** | The actual work, with commands and expected output |
| **Your 60/40 split** | What you type vs what you delegate |
| **Gotchas** | What actually breaks, and the fix |
| **Verify** | Tick list before moving on |
| **Check yourself** | 10 questions. Answer out loud. This is the real test |

---

## Read this first

**Read the chapter before you build it.** Every chapter opens with "Understand
this first" — no code in it, explaining the shape of the problem before the
solution appears. **Skipping it turns this into a copy-paste tutorial.** The
code is the easy part; the reasoning is the part that survives.

**Each chapter splits the work 60/40 in your favour:**

| Split | What goes where |
|---|---|
| **60% self-coded** | Config, the Zod contracts, the server, the schemas, the crypto, the sync protocol. These are the parts you will still be able to debug in two years. **Never paste them.** |
| **40% OpenCode** | Boilerplate, component scaffolding, repetitive classes, CI YAML, Dockerfile, test setup, refactors, error copy. |

The rule: **if OpenCode wrote it, you must be able to explain every line out
loud.** If you cannot, it was not allowed to write it.

**Every chapter ends the same way:** a 60/40 table naming the specific tasks, a
**Gotchas** list of what actually breaks, a verification checklist, and ten
questions that assume you read it. **The questions are the chapter.** If you
cannot answer them, you have not read it.

---

## Why backend first

You asked for this order and it is the right call.

A real company does not build a landing page first. It builds **the contract,
then the client**. Three reasons this order is better than the obvious one:

1. **The API is the contract.** Once `SyncEntry` exists, both sides must agree.
   Building it first means the UI is written against a real, typed interface
   instead of invented and retrofitted.
2. **You learn the fundamentals you are weakest in.** HTTP, status codes,
   databases, auth, concurrency — the exact topics that appear in interviews.
3. **It is reversible; the UI is not.** You can change the API shape in an
   afternoon. Rewriting six screens to match is a week.

**So: Chapters 3–8 are a real, running, deployed API. Chapters 9–17 are the
product that consumes it.** And the extension — the thing that actually reads
forms — arrives at Chapter 13, which means you have a working backend before you
have anything visual.

---

## The map

### Phase 1 — Foundations · Days 1–2

You cannot write code until the tools work. These two days are not
ceremonial; they are where most self-taught developers quietly break their own
setup and spend two days debugging it.

| Day | Chapter | You can prove it worked when |
|---|---|---|
| 1 | [1. Foundations](./01-foundations.md) | A TypeScript project runs and you can explain every line of its config |
| 2 | [2. The Monorepo](./02-monorepo.md) | `pnpm dev` resolves a 2-app, 6-package graph. `packages/fields` exists |

### Phase 2 — The Backend · Days 3–8 · **build this first**

This is the whole server. No UI exists yet, and that is deliberate — you can
test every endpoint with `curl` before a single component exists.

| Day | Chapter | You can prove it worked when |
|---|---|---|
| 3 | [3. What a Server Actually Is](./03-node-and-http.md) | You wrote a working server **with no framework** and can explain all 8 steps of a page load out loud |
| 4 | [4. Backend Foundation](./04-backend-foundation.md) | An API runs against MongoDB, and §11 has been rewritten to tell the truth |
| 5 | [5. The Data Model](./05-data-model.md) | Every schema written, indexed, and shown to contain no user content |
| 6 | [6. Auth](./06-auth.md) | Register, log in, rotate a token, revoke one device |
| 7 | [7. The API Reference](./07-api-reference.md) | Twelve endpoints, an error catalogue, and contract tests that fail when it drifts |
| 8 | [8. The Sync Engine](./08-sync-engine.md) | Two devices converge, offline edits survive, no edit is silently lost |

**After Day 8 you have a deployed, authenticated, tested API.** That is a real
backend, and it is what you would be asked about in an interview.

### Phase 3 — The Frontend · Days 9–18

Now you build the product that consumes it. Each chapter is real, working
product code — not stubs.

| Day | Chapter | You can prove it worked when |
|---|---|---|
| 9 | [9. The Design System](./09-design-system.md) | Tokens, base components, and provenance chips you can defend on contrast |
| 10 | [10. The Mascot](./10-the-mascot.md) | One geometry, four states, every size, zero external assets |
| 11 | [11. The Web App Shell](./11-web-app-shell.md) | Every route exists, navigates, and is honestly empty |
| 12 | [12. The Vault](./12-the-vault.md) | AES-GCM in IndexedDB. Lock, unlock, wrong passphrase, export, delete |
| 13 | [13. The Content Script](./13-the-content-script.md) | **"I found 14 fields"** on a real Google Form ← **checkpoint 1** |
| 14 | [14. The Mapping Engine](./14-the-mapping-engine.md) | 14/14 mapped by rule, each with a confidence and a reason you can read |
| 15 | [15. Fermata](./15-fermata-review-screen.md) | **One form filled end-to-end, and you pressed submit** ← **MVP** |
| 16 | [16. The Setlist & Verses](./16-the-setlist.md) | The tracker persists, Encore re-fills a stale form, Verses compresses an essay |
| 17 | [17. Document Extraction](./17-document-extraction.md) | A text PDF and a photographed marksheet both produce reviewable facts |

### Phase 4 — Ship · Days 19–21

| Day | Chapter | You can prove it worked when |
|---|---|---|
| 18 | [18. CI/CD & Deployment](./18-cicd-deploy.md) | One command ships all three surfaces, and you have run a rollback |
| 19 | [19. Production Hardening](./19-production-hardening.md) | Budgets enforced, axe clean, a screen reader walkthrough completed |
| 20 | [20. Ship Checklist](./20-ship-checklist.md) | Submitted. The privacy policy is greppable. You know what you refused to build |

### Reference

| Doc | What it holds |
|---|---|
| [`TECH-STACK.md`](./TECH-STACK.md) | Every technology, version, why it was chosen, and what it would cost to swap |
| [`../refrain.md`](../refrain.md) | The master product document. Single source of truth |
| `../refrain/` (you build it) | The actual monorepo |

---

## The two checkpoints, and they are not optional

**Stop the build, record 40 seconds of screen, send it to one person.** Momentum
comes from having shown something to a human who is not you.

| When | What you show | What it proves |
|---|---|---|
| **Day 8** | `curl` a sync round-trip: push a change, pull it back | The backend is real |
| **Day 14** | The panel saying "I found 14 fields" on a real portal | The core is real |
| **Day 16** | One form filled end-to-end. You pressed submit | It actually works |

Day 13's checkpoint is the honest one, and it is the one people skip. "I found
14 fields" is achievable with the content script and the mapping engine alone.
**If you cannot clear it, nothing later matters.** A product that scans
perfectly and cannot fill is a product that demonstrates failure — and you want
that failure on Day 13, while you can still fix it.

---

## The one rule that governs everything

From `refrain.md` §20:

```
Form Reader → Smart Mapping → Review Screen → Submit
```

**The last arrow belongs to the human, permanently.** It is not a Phase 1
limitation and it is not a roadmap item. It is structural: `filled` is
`type: "final"` and has no `SUBMIT` transition, so no code path *could* submit.

This means you will deliberately ship something **wrong in places and
incomplete** before you ship something complete. That is not a compromise.
That is the strategy.

---

## Six ideas that will save you

Everything hangs off these. Learn them properly and the rest is mechanical.

### 1. A server is a loop that waits

```
while (true) { wait for connection → handle → reply → repeat }
```

That is the whole idea. Every framework is a convenience wrapper around it, and
you write it by hand in Chapter 3. **If you can explain that loop and the event
loop that runs it, you can answer the two most common backend interview
questions there are.**

### 2. The API is the contract, and it is shared code

`packages/fields` is imported by **both** the extension and the API. One Zod
schema, one wire format. A shape change fails CI, not production.

This is also why the server is TypeScript and not Python: the schema would
otherwise be written twice, with nothing comparing the copies, and Chapter 8's
three-way merge is exactly the code that fails *silently* when two definitions
disagree.

### 3. The content script must stay dumb

It is the **only** code allowed to touch the page. No UI, no state, no business
logic. It reads a form schema and writes values. That is all.

Every product in this category breaks when a portal changes its HTML.
Isolating the fragile part means breakage is a small, isolated diff — and a
data change in `aliases.json` — rather than a rewrite.

### 4. The human gate is the product

You are not building an agent. You are building something that does the typing,
stops, and waits. That boundary is your differentiation, your marketing, and
your legal defensibility at the same time.

### 5. A server that cannot read your data cannot query your data

The hardest constraint in the product, and the one that forced the whole
backend design.

If your server holds only ciphertext, it cannot answer "which forms did I fill
last month." So search is client-side. So the sync metadata is a strictly
limited, content-free projection. So `blobs` and `syncmeta` are separate
collections with an enforced field cap.

### 6. You cannot recall a shipped extension

Chrome updates extensions silently, over days or weeks. There is no unpublish,
no hotfix, no recall.

So at every moment, real users are running **every extension version you have
ever shipped**, against your newest web app. Your newest code must work with
the oldest code you ever released. That becomes an explicit rule in Chapter 18:
schema changes are **additive-only** for one minor version.

---

## What this is not

- **Not a product spec.** `refrain.md` is that.
- **Not a copy-paste tutorial.** Every block is there to be understood and then
  typed or deleted. Nothing works if you skip the reasoning.
- **Not 20 days of typing.** It is ~16,000 lines of curriculum describing 20
  days of work. The day numbers are a **budget, not a quota** — Chapter 15 is the
  one that reliably runs over, and Chapters 13 and 17 usually run over too.

---

*Twenty days. Twenty chapters. Chapter N is Day N. Read the chapter, then build it.*
*Start with [Chapter 1 — Foundations](./01-foundations.md).*