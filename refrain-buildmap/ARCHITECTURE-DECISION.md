# Architecture Decision — where Node ends and Python begins

> **Read this before Chapter 12.** You told me you want to learn Redis, RAG,
> Qdrant, and FastAPI eventually. Two of those conflict with what we have
> built. This is the resolution, and it is a real architectural boundary
> rather than a compromise.

---

## The conflict, stated honestly

**RAG and Qdrant need to read your data. The vault's entire design exists so
that nothing can.**

| | Requires |
|---|---|
| Encrypted vault (Ch.6, Ch.13) | The server holds **ciphertext it cannot decrypt**. Ever. |
| RAG over your documents | Something must **read the text**, chunk it, and call an embedding model. |
| Qdrant | Needs the **plaintext vectors and the text**, stored in a database the server can query. |

So "add RAG to my encrypted vault" is not a feature request. It is a request to
undo the central security property of the product.

**And the reason matters:** Refrain's entire pitch is that a student's
marksheet never leaves their device. Add a cloud embedding service and that
claim is false. Not "weakened" — false. And Chapter 19's privacy policy, the
Web Store disclosure, and your README would all need rewriting.

---

## The resolution: two tiers, two threat models

This is how real products do it, and it gives you **both** things without
lying to anyone.

```
┌─────────────────────────────────────────────────────────────┐
│ TIER 1 — LOCAL. The vault. Never leaves the device.         │
│                                                             │
│   passphrase ─PBKDF2─> k_root ─HKDF─> k_vault (never sent)  │
│   k_vault ─> per-document DEK ─> AES-GCM ciphertext          │
│                                                             │
│   Encrypted at rest in IndexedDB. Server never sees it.      │
│   THIS IS THE PRODUCT. Chapters 6, 13.                       │
└─────────────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────────────┐
│ TIER 2 — OPTIONAL AI SERVICE. Explicitly consented.         │
│                                                             │
│   User opts in ON PURPOSE for ONE document.                 │
│   That document is decrypted on-device, sent to *your*       │
│   service, embedded, stored in Qdrant, queried.             │
│                                                             │
│   Scoped to what the user chose. Revocable. Auditable.       │
│   Clearly disclosed — NOT "nothing leaves your device".      │
└─────────────────────────────────────────────────────────────┘
```

### What this buys you

- **Tier 1 stays honest.** The marketing claim survives intact, because it
  was always scoped to the vault.
- **Tier 2 is where every technology you named lives.** Redis, Qdrant,
  embeddings, RAG, FastAPI — all of it, in a service that is *allowed* to read
  its inputs.
- **The interview story is better, not worse.** "How do you ship end-to-end
  encryption *and* AI search over the same user's data?" is a senior question
  with a real answer. "We put everything in the cloud" is not.

### The cost, stated plainly

You now operate **two systems with different security properties**, and the
disclosure text gets longer. That is real work and real liability. It is also
what every serious AI product does, which is why you can read how they do it.

---

## Node or Python? Now that the boundary is drawn, it is easy.

| Tier | Language | Why |
|---|---|---|
| **Tier 1** — extension, vault, content script, web app | **TypeScript** | Shares Zod contracts with the server. `crypto.subtle` is the same API in the browser and in Node. Chrome MV3 is JavaScript-only, so this was never optional. |
| **Tier 2** — AI service | **Python** | FastAPI + Qdrant + the embedding ecosystem are all Python-first. Fighting that in TypeScript is unpaid labour. |

**You were right the first time.** When you asked "why not Python or Fast API,"
your instinct was pointing at exactly this — you just asked it before the
product had a Tier 1 to protect.

I gave you Node. That answer was correct *for Tier 1* and I should have told
you it was scoped, because your actual goal was a broader one.

---

## What changes in the build order

| Chapters | Change |
|---|---|
| 1–11 (frontend) | **Unchanged.** Still the shippable product. Still needs no server. |
| 12–16 (sync backend) | **Optional and lower priority.** You said you want to learn Redis/RAG — that is Tier 2, which is *more* valuable to you than sync. Consider deferring. |
| 17–19 (ship) | Unchanged. |

### The reordering I would actually recommend

Phase 1 stays exactly as it is — 15 days to a shippable product. That is
unchanged and still correct.

Phase 2 becomes a choice between two different second acts:

| Option | What you build | What you learn |
|---|---|---|
| **A. Sync first** (chapters as written) | E2EE sync across devices | OAuth, JWT, conflict resolution, MongoDB, migrations. **Solid backend interview material.** |
| **B. AI service first** | Tier 2: FastAPI + Qdrant + RAG + Redis cache | Embeddings, vector search, retrieval, caching, prompt engineering. **Solid AI-product interview material.** |

**You told me your weakness is interview fundamentals.** By that measure,
Option B is the higher-value second act — AI-product interviews ask about
embeddings, chunking, retrieval quality, and caching far more often than they
ask about JWT rotation.

But Option A teaches the harder fundamentals, and "harder" correlates with
"more likely to be asked."

**My recommendation: build both, but write Tier 2 first.** Here is why that is
possible — Tier 2 has *no dependency* on the sync backend. It needs a place to
receive an opted-in document, not a sync engine, not accounts, not a MongoDB
cluster. You can build a working Qdrant-backed RAG service in **two days**
against local files, and learn the entire retrieval stack, before you have
written a single line of the sync backend.

---

## The technology list, and when you meet it

You named these. Here is where each one actually enters the project, so you
know what you are learning *and* why.

| Technology | Chapter | What it teaches |
|---|---|---|
| **Node.js** | 1, 12 | The runtime your whole frontend shares |
| **TypeScript** | 1, 2 | Type systems, generics, structural typing |
| **pnpm / workspaces** | 2 | Dependency management, monorepos |
| **Turborepo** | 2 | Build graphs, caching, task orchestration |
| **React** | 3, 5 | Components, hooks, rendering |
| **Tailwind** | 3 | Design systems, tokens, utility CSS |
| **WebCrypto** | 6 | AES-GCM, PBKDF2, HKDF — *real* crypto, not a library |
| **Dexie / IndexedDB** | 6 | Client-side storage, indexes, transactions |
| **PDF.js / Tesseract** | 11 | Document parsing, OCR, canvas |
| **XState** | 9 | State machines, guards, modelling UI state |
| **Playwright** | 7, 9 | E2E testing, selectors, flakiness |
| **Hono** | 12 | HTTP, middleware, Web-standard APIs |
| **MongoDB / Mongoose** | 13 | Schemas, indexes, aggregation, ODM |
| **JWT / Ed25519** | 14 | Auth, key pairs, token rotation |
| **Redis** | *Tier 2* | Caching, rate limiting, queues |
| **Qdrant** | *Tier 2* | Vector search, embeddings, ANN indexes |
| **RAG** | *Tier 2* | Chunking, retrieval, grounding, evaluation |
| **FastAPI** | *Tier 2* | Python APIs, type hints, Pydantic |

---

## The one thing I want you to hold onto

You said you could not answer interview questions a year into web development.

**That is not a knowledge problem. It is a retrieval problem.**

Nobody retains everything they read. What separates someone who can answer
"why AES-GCM and not AES-CBC?" is not having memorised it — it is having
*once written the line* `AES-GCM` in a file, hit the exact error it caused, and
had to explain it to themselves at 1am.

That is why this project is built the way it is. The three bugs in Chapter 7
produce an identical, unhelpful symptom on purpose — because the fix only
sticks if you understand the mechanism, not the fix. And it is why every file
in `packages/` carries a comment explaining *why that line is there* rather
than restating what it does.

So when you finish Chapter 6, you will not "know AES-GCM." You will have
debugged an IV collision and a tamper-detection failure, and that is a
different thing entirely.

---

**Now decide: do we build the sync backend (chapters 12–16) next, or jump to
the Tier 2 AI service and learn Redis/Qdrant/RAG/FastAPI?**

My vote is Tier 2 first. It is faster to a working demo, it is closer to what
you want to learn, and it does not touch the frontend you have already built.