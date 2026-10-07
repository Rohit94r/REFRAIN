# Fundamentals — the layer you read *before* the chapters

> **These are the interview topics.** The 19 chapters teach you a project.
> These 8 teach you the ideas the chapters assume you already have.
>
> Read them in order. Each one is short, and each one ends with questions
> you should be able to answer out loud.

---

## Why this layer exists

You said you finished a web development course a year ago, built a project
that was entirely AI-generated, and then could not answer interview questions.

That is not a failure to study. It is a specific and very common failure mode:

**Reading about something is not the same as having built it.**

You can watch a video on AES-GCM and understand it. You cannot explain why the
IV must never be reused until you have hit the bug that a reused IV creates.
Interview questions do not ask "what is AES-GCM." They ask "a user reports
their vault fails to open on one device but not another — what do you check?"

That question is only answerable if you built something that could fail.

So the structure is deliberately inverted from a course:

| | A course | This |
|---|---|---|
| Order | Concepts, then project | Project, with concepts **in place** |
| Failure | You get it right | You get it **wrong first**, then find out why |
| Retention | You remember what you read | You remember what broke |
| Interview | You recognise questions | You have **been through them** |

**Read the chapter, build it, get it wrong, read the Gotchas section, fix it,
then read the fundamentals doc for that chapter.** The fundamentals doc is not
a prerequisite. It is the explanation you want *after* you have hit the wall.

---

## The eight documents

| Doc | Covers | Read it after |
|---|---|---|
| [01 — JavaScript that matters](./01-javascript-that-matters.md) | Closures, `this`, prototypes, async, the event loop | Ch.1 |
| [02 — TypeScript that matters](./02-typescript-that-matters.md) | Structural typing, generics, narrowing, `unknown` | Ch.2 |
| [03 — HTTP, and why extensions can](./03-http-and-the-browser.md) | Requests, CORS, same-origin, cookies, CSP | Ch.5, 7 |
| 04 — Node, and what it actually is | Event loop, modules, why Node vs Python | ⬜ write at Ch.12 |
| [05 — Encryption, from first principles](./05-encryption-first-principles.md) | Hashing, symmetric, AEAD, nonces, key derivation | **Ch.6** |
| 06 — Databases, relational vs document | Schemas, indexes, normalisation, aggregation | ⬜ write at Ch.13 |
| 07 — Caching, queues, and Redis | Cache-aside, invalidation, TTLs, rate limits | ⬜ write at Tier 2 |
| 08 — Retrieval, embeddings, and RAG | Chunking, vectors, Qdrant, RAG quality | ⬜ write at Tier 2 |

Documents 04, 06, 07 and 08 are written **the day you need them** — at Chapter 12,
Chapter 13, and Tier 2 respectively. They are listed now so you know they are
coming and what they will cover.

---

## The interview question types, and where each is answered

Because you named interview-readiness as the actual goal, here is the map.
Every question type traces to a specific document and a specific chapter — so
when you are asked, you can point at the thing you actually built.

| If they ask about | Read | You will have built |
|---|---|---|
| Closures, `this`, event loop | `learn/01` | Every React component |
| Generics, type narrowing | `learn/02` | `packages/fields` |
| CORS, same-origin policy | `learn/03` | The content script |
| Why a content script and not a web app | `learn/03` + Ch.7 | The extension itself |
| Auth, tokens, session vs JWT | Ch.14 + `learn/05` | Argon2id, key rotation |
| Encryption choices | `learn/05` | PBKDF2, HKDF, per-doc DEKs |
| Indexes, query performance | `learn/06` | Dexie indexes, Mongo indexes |
| Schema design, denormalising | `learn/06` | `blobs` / `syncmeta` split |
| Caching, invalidation | `learn/07` | Redis in the AI service |
| Rate limiting | `learn/07` + Ch.16 | The API limiter |
| RAG, embeddings, chunking | `learn/08` | The Qdrant service |
| System design, scaling | `learn/08` + Ch.18 | Two tiers, PITR, load tests |

---

## How to actually use this (do not skip this part)

### The loop

For each chapter:

```
1. Read "Understand this first" in the chapter      ← 10 min
2. Build it, typing everything                      ← the work
3. Get it wrong. Fix it from the Gotchas section.   ← where learning happens
4. Read the matching fundamentals doc               ← now it means something
5. Answer the 10 "Check yourself" questions OUT LOUD ← the real test
```

**Step 5 is the one that matters and the one everyone skips.** Reading a
question and recognising the answer is not the same as producing it from
nothing. Say the answer aloud, badly, then look. If you cannot produce it,
you have found the gap — and now you know exactly which document to read,
instead of vaguely knowing you "need to revise."

### The rules

**Never paste code you do not understand.** If OpenCode wrote it and you cannot
explain it, delete it and write it yourself. That is not purity — it is that
the interview will not ask about the parts you delegated.

**One commit per concept, not per day.** A commit whose message explains *why*
is future documentation. Chapter 2 sets this up.

**Read error messages.** Every single one. Not "TS2345, add a string." Read
what it says the code was and what was expected. The error is the tutor.

**Keep a `NOTES.md`.** Every question you could not answer, and what fixed it.
That file is your revision list for the week before an interview, and nothing
else you write will be as useful.

---

## What "done" looks like

You are not done when the product ships. You are done when you can sit down
and, unprompted, answer:

1. Why can a browser extension read a form but a web app cannot?
2. What is AES-GCM and what would go wrong with plain AES-CBC?
3. Why must an IV never be reused with the same key?
4. What does your key hierarchy look like, and why is it not one key?
5. What happens if a user's phone clock is wrong?
6. Why is your server unable to search your users' data?
7. What is an index, and what happens when you drop one?
8. How would you add search over documents you cannot read?
9. What is a race condition, and where is one in this codebase?
10. Why does `USER_EDITED` not advance the mascot's state?

If you can answer those ten cold, you are interview-ready — and you will know
it, rather than hoping.

**That list is the point of the whole project.** Everything else is a means of
getting there.

---

*Start with [01 — JavaScript that matters](./01-javascript-that-matters.md).*
*Then Chapter 1.*