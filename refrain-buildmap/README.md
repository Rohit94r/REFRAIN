# Refrain — Build Map

> **Answer once. Fill everywhere.**
>
> Nineteen chapters, in order, from `pnpm init` to a submitted Chrome extension and a deployed
> sync API. Frontend first, then backend.

`refrain.md` is the product spec. **This folder is the curriculum** — the order to build in, the
concept behind each step, the exact commands, and the reasoning for every architectural choice
the master document makes.

---

## Read this before you start

**Read the chapter before you build it. Not after.** Every chapter opens with "Understand this
first" — a section with no code in it, explaining the shape of the problem before the solution
appears. **Skipping it turns this into a copy-paste tutorial.** The code is the easy part; the
reasoning is the part that survives.

**Each chapter splits the work 60/40 in your favour:**

| Split | What goes where |
|---|---|
| **60% self-coded** | Config, the Zod contracts, the content script, the mapping rules, the crypto, the sync protocol, the privacy policy. These are the parts you will still be able to debug in two years. Never paste them. |
| **40% OpenCode** | Boilerplate, component scaffolding, repetitive Tailwind classes, CI YAML, Dockerfile, test setup, refactors, error copy. |

The rule that makes this work: **if OpenCode wrote it, you must be able to explain every line of
it out loud.** If you cannot, it was not allowed to write it.

**Every chapter ends the same way:** a 60/40 table naming the specific tasks, a "Gotchas" list
of what actually breaks, a verification checklist, and ten questions that assume you read it.
**The questions are the chapter.** If you cannot answer them, you have not read it, and the code
you are about to write is guesswork.

---

## The map

### Phase 1 — Frontend, Days 1–15 · shippable

**Fifteen days, and at the end of Day 15 you have a complete, useful, server-free product** that
you could submit on Day 16. Every phase boundary in this map is chosen so a working thing
exists on the other side of it.

| Day | Chapter | You can prove it worked when… |
|---|---|---|
| 1 | [1. Foundations](./01-foundations.md) | A TypeScript project runs and you can explain every line of its config |
| 2 | [2. The Monorepo](./02-monorepo.md) | `pnpm dev` resolves a 2-app, 6-package graph. `packages/fields` exists |
| 3 | [3. The Design System](./03-design-system.md) | Tokens, base components, and provenance chips you can defend on contrast |
| 4 | [4. The Mascot](./04-the-mascot.md) | One geometry, four states, every size, zero external assets |
| 5 | [5. The Web App Shell](./05-web-app-shell.md) | Every route exists, navigates, and is honestly empty |
| 6 | [6. The Vault](./06-the-vault.md) | AES-GCM in IndexedDB. Lock, unlock, wrong passphrase, export, delete |
| 7–8 | [7. The Content Script](./07-the-content-script.md) | **"I found 14 fields"** on a real Google Form ← **checkpoint 1** |
| 9–10 | [8. The Mapping Engine](./08-the-mapping-engine.md) | 14/14 mapped by rule, each with a confidence and a reason you can read |
| 11–13 | [9. Fermata](./09-fermata-review-screen.md) | **One form filled end-to-end, and you pressed submit** ← **MVP** |
| 14 | [10. The Setlist & Verses](./10-the-setlist.md) | The tracker persists, Encore re-fills a stale form, Verses compresses an essay |
| 15 | [11. Document Extraction](./11-document-extraction.md) | A text PDF and a photographed marksheet both produce reviewable facts |

### Phase 2 — Backend, Days 16–23 · optional

**None of this is required to ship.** Chapter 19 submits v1.0 without a single line of it. Read
[Chapter 12's opening section](./12-backend-foundation.md) before deciding whether to start.

| Day | Chapter | You can prove it worked when… |
|---|---|---|
| 16 | [12. Backend Foundation](./12-backend-foundation.md) | An API runs against MongoDB — and §11 has been rewritten to tell the truth |
| 17 | [13. The Data Model](./13-data-model.md) | Every schema written, indexed, and shown to contain no content |
| 18 | [14. Auth](./14-auth.md) | Register, log in, rotate a token, revoke one device |
| 19 | [15. The Sync Engine](./15-sync-engine.md) | Two devices converge, offline edits survive, no edit is silently lost |
| 20 | [16. The API Reference](./16-api-reference.md) | Twelve endpoints, an error catalogue, and contract tests that fail when it drifts |
| 21 | [17. CI/CD & Deployment](./17-cicd-deploy.md) | One command ships all three surfaces, and you have run a rollback |
| 22 | [18. Production Hardening](./18-production-hardening.md) | Budgets enforced, axe clean, a screen reader walkthrough completed |
| 23 | [19. Ship Checklist](./19-ship-checklist.md) | Submitted. The privacy policy is greppable. You know what you refused to build |

### Reference

| Doc | What it holds |
|---|---|
| [`TECH-STACK.md`](./TECH-STACK.md) | Every technology, version, why it was chosen, and what it would cost to swap |
| [`../refrain.md`](../refrain.md) | The master product document. 23 sections. Single source of truth |
| `refrain/` (you create it) | The actual monorepo |

---

## Two checkpoints, and they are not optional

**Stop the build, record 40 seconds of screen, send it to one person.** Momentum comes from
having shown something to a human who is not you.

| When | What you show | What it proves |
|---|---|---|
| **Day 8** | The panel saying "I found 14 fields" on a real portal | The core is real |
| **Day 13** | One form filled end-to-end. You pressed submit | It actually works |

Day 8's checkpoint is the honest one, and it is the one people skip. "I found 14 fields" is
achievable with the content script and the mapping engine alone. **If you cannot clear it,
nothing later matters.** A product that scans perfectly and cannot fill is a product that
demonstrates failure — and you want that failure on Day 8, while you can still fix it.

---

## The one rule that governs everything

From `refrain.md` §20:

```
Form Reader → Smart Mapping → Review Screen → Submit
```

**The last arrow belongs to the human, permanently.** It is not a Phase 1 limitation and it is
not a Phase 2 roadmap item. It is structural: `filled` is `type: "final"` and has no `SUBMIT`
transition, so no code path *could* submit.

This means you will deliberately ship something **wrong in places and incomplete** before you
ship something complete. That is not a compromise. That is the strategy. Phase 2 is where it
becomes good; Phase 1 is where it becomes *real*.

---

## Six ideas that will save you

Everything in this map hangs off these. Learn them properly and the rest is mechanical.

### 1. You cannot build a web app that fills other people's forms

Not a bug. **The same-origin policy.** A page at `refrain.dev` has literally zero access to the
DOM of `forms.google.com`.

Exactly four things can read another site's form: a **browser extension**, a userscript, a
bookmarklet, or a native app. Extensions win on every axis — install friction, permission
model, ability to run in the background, store distribution.

→ **Chapter 7** is where this stops being theoretical and you read your first cross-origin DOM.

### 2. Content scripts must stay dumb

The content script is the **only** code allowed to touch the page. No UI, no state, no business
logic. It reads a form schema and writes values. That is all.

Every product in this category breaks when a portal changes its HTML. Isolating the fragile
part means breakage is a small, isolated, fixable diff — and a data change in `aliases.json` —
rather than a rewrite.

→ **Chapter 7.** Do not "improve" that file. Resist. The three bugs it teaches you will all look
like the same symptom, which is why they are taught together.

### 3. The human gate is the product

You are not building an agent. You are building something that does the typing, stops, and
waits. That boundary is simultaneously your differentiation, your marketing, and your legal
defensibility.

`grep` your own tree for a CAPTCHA bypass before you ship. If it is there, you have built a
different, worse, unshippable product.

→ **Chapter 9** builds Fermata, the hero screen. Spend your disproportionate time here.

### 4. Local-first is arithmetic, not virtue

Local inference costs you about **₹0 per fill**. Competitors meter inference per action. A
student filling 60 forms a month **destroys their unit economics** — they have to keep users on a
slow, expensive path to survive.

That is the moat, and it is made of arithmetic rather than features. Which means **every time you
consider adding a cloud call, you are spending the moat.**

→ **Chapter 8** wires local inference with the fallback chain.

### 5. A server that cannot read your data cannot query your data

This is the single hardest constraint in the product, and it is the one that forced the whole
backend design.

If your server holds only ciphertext, it cannot answer "which forms did I fill last month" for
you, and it certainly cannot answer it for the user. So search is client-side. So the sync
metadata is a strictly limited, content-free projection. So `blobs` and `syncmeta` are separate
collections with an enforced field cap.

→ **Chapter 13** is where the hybrid storage model is derived from this sentence.

### 6. You cannot recall a shipped extension

Chrome updates extensions silently, in the background, over days or weeks. There is no
unpublish, no hotfix, no recall.

So at every moment, real users are running **every extension version you have ever shipped**,
against your newest web app. Your newest code must work with the oldest code you ever released.

→ **Chapter 17** turns that into an explicit rule: schema changes are **additive-only** for one
minor version. Renaming a field breaks 22% of your users for two weeks and you cannot fix it.

---

## What this is not

- **Not a product spec.** `refrain.md` is that.
- **Not a copy-paste tutorial.** Every code block is there to be understood and then typed or
  deleted. Nothing here works if you skip the reasoning.
- **Not 15 days of typing.** It is ~11,900 lines of curriculum describing 23 days of work. The
  day numbers are a **budget, not a quota** — some chapters are two hours, Chapter 9 is three
  days, and Chapters 7 and 11 are the two that reliably run over. If you are on Day 20 of your
  own calendar, you are not behind.

> **Why 23 days when you asked for 15?** Because the fifteen days are all in Phase 1, and
> Phase 1 ends with a **complete, shippable, server-free product**. Phase 2 is the backend —
> eight chapters that Chapter 19 explicitly tells you *not* to ship in v1.0. You prove demand
> with the no-server version first, then build sync for the people who asked for it. The backend
> is waiting for a demand signal, not a deadline.

---

*Twenty-three days. Nineteen chapters. Read the chapter, then build it.*