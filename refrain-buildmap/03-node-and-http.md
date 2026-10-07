# Chapter 3 — What a Server Actually Is

> **Day 3 · Goal: you can explain, from memory, what happens when someone types
> `refrain.dev` into a browser — and you have written a server that does it.**
>
> **This is the most important chapter in the book.** Everything after it is a
> variation on what you learn here. We build the backend first, exactly like a
> real company does, because the API is the contract and the UI is a consumer
> of it.

---

## Understand this first

### Most people cannot explain this. That is the gap you have.

Type this into a browser:

```
https://refrain.dev/settings
```

Here is what actually happens, in order:

```
1.  You type "refrain.dev/settings"
2.  The browser checks: is this a name I recognise?
        NO → asks a DNS server "what IP address is refrain.dev?"
             DNS says 76.76.21.21
3.  The browser opens a TCP connection to 76.76.21.21:443
4.  TLS handshake. The server proves it is really refrain.dev and nobody is
    reading. Both sides agree on a session key.
5.  The browser sends an HTTP request:
        GET /settings HTTP/1.1
        Host: refrain.dev
        Accept: text/html
6.  A SERVER receives it. It might be Node, Python, Go, or a CDN edge.
    It routes /settings to some code.
7.  That code returns a response:
        HTTP/1.1 200 OK
        Content-Type: text/html
        <!doctype html>...
8.  The browser renders the HTML, then requests /assets/app.js, renders that,
    requests the CSS, and you finally see a page.
```

**Every question you were unable to answer fits somewhere in those 8 steps.**

If someone asks "what happens on a page load," and you can walk those steps
without hesitating, you have answered it. That is the whole skill.

### What a "server" actually is

This is where most tutorials hand-wave. It is simpler than it sounds.

**A server is a program that does exactly two things, forever:**

```ts
// THIS IS A SERVER. That is the whole idea.
while (true) {
  const socket = await waitForIncomingConnection()
  const request = await readHttpRequest(socket)
  const response = handleRequest(request)      // ← your code goes here
  await writeHttpResponse(socket, response)
}
```

That is genuinely all. **"Server" is not a different kind of program.** It is a
program that waits for input instead of exiting. Your CLI script that reads
stdin and prints is a server in every way that matters — it just stops when the
input ends.

The two things that make it feel like a different world:

| | Script | Server |
|---|---|---|
| Ends when | The work is done | **Never** |
| Waits for input? | No, it has its arguments | **Yes, indefinitely** |
| Handles many users? | One | **Thousands, concurrently** |

**Concurrency is the part that surprises people.** Your `handleRequest` is
called for user A, then user B, then user C. If request A takes 3 seconds and
you block on it, users B and C wait. That is why async exists — and why
Chapter 4's `async` handler is not decoration.

### Node is not "a server framework." It is this loop, built in.

```ts
// The rawest possible server in Node. No framework. This is the idea.
import { createServer } from "node:http"

createServer((req, res) => {
  res.writeHead(200, { "Content-Type": "text/html" })
  res.end("<h1>Hello from a server</h1>")
}).listen(3000)
```

Run it, open `localhost:3000`, and you have understood servers. Everything
else — Hono, Express, FastAPI — is a convenience wrapper that eventually does
this.

> **This is the answer to "what's the difference between Express and Node?"**
> Node has a server built in. Express is a layer that makes routing, parsing
> and middleware easier. That is the entire difference.

---

## Step 1 — Build a server with nothing but Node

Do this before touching a framework. If you can do this, every framework is
just a shortcut.

```bash
mkdir -p ~/refrain-server && cd ~/refrain-server
npm init -y
node --version   # v26
```

### `server.js` — a server you fully understand

```js
import { createServer } from "node:http"
import { readFile } from "node:fs/promises"

/**
 * THE WHOLE IDEA, IN ONE FILE.
 *
 * A server waits. When something connects, it runs `handler`. That is it.
 * Everything else — routing, middleware, JSON parsing — is a library
 * making this loop pleasant to write.
 */
const server = createServer(async (req, res) => {
  // `req` is what the client sent. `res` is how you reply.
  console.log(`${req.method} ${req.url}`)

  // ── Routing ───────────────────────────────────────────────────
  // This is what Express and Hono automate.
  if (req.url === "/api/health") {
    res.writeHead(200, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ ok: true, ts: Date.now() }))
    return
  }

  if (req.url === "/") {
    res.writeHead(200, { "Content-Type": "text/html; charset=utf-8" })
    res.end("<h1>Refrain</h1><p>Your answer once. Fill everywhere.</p>")
    return
  }

  // ── 404: a real status code, not a 200 with "not found" ────────
  // Getting this wrong is a genuine bug. Search engines, caches, and
  // monitoring all treat a 200-with-error-body as success.
  res.writeHead(404, { "Content-Type": "application/json" })
  res.end(JSON.stringify({ error: "not_found" }))
})

// The 3-argument form is the difference between Node and Express.
// The third argument runs BEFORE your handler, on every request.
// That is "middleware" — and it is the single most important idea
// in every backend framework.
const PORT = 3000
server.listen(PORT, () => {
  console.log(`listening on http://localhost:${PORT}`)
})
```

```bash
node server.js
```

```bash
# In another terminal — test it properly, with the status codes shown
curl -i http://localhost:3000/api/health
curl -i http://localhost:3000/
curl -i http://localhost:3000/nope
```

**What you should see:**

```
HTTP/1.1 200 OK
Content-Type: application/json

{"ok":true,"ts":1757248800000}

HTTP/1.1 404 Not Found
{"error":"not_found"}
```

> **`curl -i`, not `curl`.** The `-i` prints response headers, which is where
> the status code lives. Without it you get only the body and cannot tell a
> 404 from a 200. This is the single habit that makes HTTP click.

### The request object — what actually arrives

Print it once and look:

```js
server = createServer((req, res) => {
  console.log({
    method: req.method,        // "GET" | "POST" | "PUT" | "DELETE"
    url: req.url,              // "/api/health?debug=1"  ← query string included!
    headers: req.headers,      // lowercased object of request headers
  })
  res.writeHead(200).end("ok")
})
```

Three things that surprise everyone:

1. **`req.url` includes the query string.** `?debug=1` is part of it. You must
   parse it.
2. **Headers are lowercase.** Always. `Content-Type` arrives as
   `content-type`.
3. **`GET /api/health` and `GET /api/health?x=1` are different strings.** So
   `if (req.url === "/api/health")` fails the moment anyone adds a query
   parameter. This bug is in every hand-rolled router ever written.

---

## Step 2 — Reading a request body, and why it is async

A `POST` body arrives in chunks. You cannot read it synchronously — **you do
not know its size yet.**

```js
server = createServer((req, res) => {
  if (req.method !== "POST") {
    res.writeHead(405, { Allow: "POST" })   // 405, not 404 — the route exists
    res.end()
    return
  }

  let body = ""

  // 'data' fires many times. 'end' fires once, when it is finished.
  req.on("data", (chunk) => {
    body += chunk
  })

  req.on("end", () => {
    const parsed = JSON.parse(body)     // now it is safe
    res.writeHead(201, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ id: "doc_1", ...parsed }))
  })
})
```

```bash
curl -i -X POST http://localhost:3000/api/docs \
  -H "Content-Type: application/json" \
  -d '{"name":"refrain","encrypted":true}'
```

### Why this matters: the size limit you did not write

```js
req.on("data", (chunk) => {
  body += chunk
  // ⚠️ A client can send 10GB. You will read all of it into memory
  //    and crash the process. Chapter 4 caps this with a 12MB limit,
  //    and Chapter 17 hits this for real on a 40MB blob upload.
})
```

**Every production server caps body size.** Ours is 12MB, because our largest
legitimate upload is a 10MB document. That is Chapter 4's job. You are meeting
the problem today so it is not a surprise later.

---

## Step 3 — Middleware, the idea every framework is built on

Refactor the body-reading into a reusable wrapper. You have just written your
first middleware.

```js
/**
 * Middleware = a function that runs before your handler, on every
 * request, and can end the request early.
 *
 * Express calls these `app.use(...)`. Hono calls them `app.use(...)`.
 * FastAPI calls them dependencies. All three are this.
 */
function jsonBody(req, res, next) {
  let body = ""
  req.on("data", (c) => { body += c })
  req.on("end", () => {
    try {
      req.body = body ? JSON.parse(body) : {}
      next()                      // continue to the handler
    } catch {
      // A malformed body is the CLIENT's fault. 400, not 500.
      res.writeHead(400, { "Content-Type": "application/json" })
      res.end(JSON.stringify({ error: "invalid_json" }))
      // Note: no next(). The request ends here. That is allowed and
      // is how middleware rejects a request.
    }
  })
}

function logger(req, res, next) {
  const t0 = Date.now()
  res.on("finish", () => {
    // 'finish' fires after the response is fully sent.
    console.log(`${req.method} ${req.url} → ${res.statusCode} in ${Date.now() - t0}ms`)
  })
  next()
}

server = createServer((req, res) => {
  logger(req, res, () => jsonBody(req, res, () => {
    // the real handler
    res.writeHead(200, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ received: req.body }))
  }))
})
```

**Two responses are a real bug.** If a middleware sends a response *and* calls
`next()`, the client gets two bodies and hangs or shows garbage. Either you
respond, or you pass through. Never both.

---

## Step 4 — Status codes you must know cold

This is memorisation, not reading. Put it in your `NOTES.md`.

| Code | Name | Use it when |
|---|---|---|
| **200** | OK | Success, and here is the body |
| **201** | Created | A POST made something new |
| **204** | No Content | Success, deliberately empty body |
| **301 / 308** | Moved Permanently | Redirect that keeps the method |
| **400** | Bad Request | Malformed — bad JSON, missing field |
| **401** | **Unauthorized** | "I don't know who you are." **Retry with new credentials.** |
| **403** | **Forbidden** | "I know who you are. No." **Do not retry.** |
| **404** | Not Found | No such route or resource |
| **405** | Method Not Allowed | Route exists, wrong verb |
| **409** | Conflict | Concurrent modification |
| **413** | Payload Too Large | Over your size cap |
| **422** | Unprocessable | Syntax fine, semantics wrong (Zod validation) |
| **429** | Too Many Requests | Rate limited |
| **500** | Internal Server Error | Your bug |

### 401 vs 403 — asked in nearly every auth interview

**401 means "I do not know who you are."** The client can get a new token and
try again.

**403 means "I know exactly who you are and you are not allowed."** Retrying
with the same credentials can never work.

```ts
// ❌ THE INFINITE LOOP
async function refresh() {
  const r = await fetch("/auth/refresh")
  if (r.status === 401) {
    await refresh()      // self-revocation returns 401. Forever. Tab dies.
  }
}

// ✅ Correct
if (r.status === 403) return null   // revoked → log out, do not retry
if (r.status === 401) return refresh()  // expired → retry ONCE
```

> Refrain has **real self-revocation** — Chapter 6 lets a user sign out of one
> device. That device's refresh token is revoked, so its next refresh **must**
> return 401. So a client that retries 401 loops forever.
>
> **Chapter 6's code has this bug on purpose, and Chapter 7's contract test
> catches it.** You cannot learn this from reading. You have to hit it.

### The rule for errors

**Never send a 200 with an error body.** `{ ok: false }` at status 200 breaks
every cache, every monitoring dashboard, and every client retry policy.

---

## Step 5 — The event loop, and why `async` exists

Your server handled one request fine. Now make it handle 5,000.

```js
// ❌ Blocks everything. One slow request stops all users.
server = createServer(async (req, res) => {
  const data = await someSlowDatabaseCall()   // 500ms
  res.end(data)
})
```

That `await` **does not block** — it yields. That is the entire point, and it
is worth understanding properly because it is asked constantly.

Node runs your JavaScript on **one thread**. When you `await`, Node:

1. Pauses this function
2. Moves to the next request
3. Comes back when the I/O finishes

```
Request A        Request B        Request C
   │                │                │
   ├──await────┐    ├──await────┐    ├──await────┐
   │           ▼    │           ▼    │           ▼
   │        (idle)   │        (idle)   │        (idle)
   │                │                │
   │◀──done─────────│◀──done─────────│◀──done─────────
   ▼
responds
```

**Three requests are in flight at once, on one thread**, because they are
waiting on I/O rather than burning CPU.

### Where this breaks: CPU-bound work

```js
// ❌ This blocks ALL other requests for 4 seconds
server = createServer(async (req, res) => {
  const hash = await crypto.subtle.digest("SHA-512", hugeBuffer)
})
```

`crypto.subtle` and I/O are different: WebCrypto runs on a **thread pool**, so
that one is fine. But a big `Array.sort()` or a regex over 50MB of text is CPU
work on the main thread, and it blocks everything.

**This is why Refrain runs OCR in a Web Worker** in the extension, and why a
production server would offload heavy CPU work to a queue or a worker thread.

> **The interview answer:** "Node is single-threaded for JavaScript with a
> libuv threadpool for I/O and WebCrypto. So it's excellent at many concurrent
> I/O-bound requests and bad at CPU-bound work — a big sort blocks every other
> user. Async lets us handle thousands of waiting requests, not thousands of
> simultaneous computations."

---

## Step 6 — What you just built, and what comes next

You wrote, with no framework:

- A working HTTP server
- Routing, by hand
- A 404 and a 405 with correct codes
- Request body parsing
- Two pieces of middleware
- Logging with timing

**Everything in Chapters 4–8 is this, plus a database.**

| What you did by hand | What the framework gives you |
|---|---|
| `if (req.url === "/api/health")` | `app.get("/api/health", handler)` |
| `req.on("data", ...)` | `await c.req.json()` |
| Middleware chains | `app.use(logger)` |
| `createServer` + `listen` | Same — that part is still Node |

**Hono is a router, not a server.** Underneath, `app.fetch()` and
`@hono/node-server` are exactly the `createServer` loop you just wrote. That
is worth knowing, because it means a framework is a convenience, not a
dependency on magic.

---

## Your 60/40 split for this chapter

**Every task is yours.** This is the chapter where the framework does not exist
yet, so there is nothing to delegate.

| # | Task | Who |
|---|---|---|
| 1 | Write `server.js` **without a framework**, from the code above | **You** |
| 2 | Run it. `curl -i` every route. Read every header | **You** |
| 3 | Print `req` and write down what is actually in it | **You** |
| 4 | Add a `POST` route and break the JSON on purpose | **You** |
| 5 | Write the middleware chain yourself | **You** |
| 6 | **Explain all 8 steps of a page load out loud, from memory** | **You** |
| 7 | Explain what a server is in one sentence | **You** |

> **This chapter is the foundation of the interview skill you said you were
> missing.** If you can walk those 8 steps and explain the event loop, you can
> answer "what happens on a page load" and "why async" — two of the most
> common questions in any backend interview — cold.
>
> Everything else in this book is easier once these two are solid.

---

## Gotchas in this chapter

**`curl` shows the body but you cannot see the status.** Use `curl -i`.

**`req.url === "/api/health"` fails when a query string is added.** It becomes
`/api/health?debug=1`. Compare a parsed pathname, not the raw URL.

**Sending a response twice hangs the client.** Middleware either responds *or*
calls `next()`. Never both.

**`req.body` is `undefined` if you never collected it.** Node does not parse
bodies. That is not a Node bug; there is no built-in parser and that is
intentional.

**A `POST` handler that returns 200 for a creation.** It is 201.

**Your server is single-threaded and one heavy request froze everything.** See
Step 5. This is the event loop, not a bug in your code.

**`JSON.parse` on the body threw and took the server down.** Wrap it. A
malformed body is the client's fault and deserves a 400, not a crash.

---

## Verify before moving on

- [ ] `server.js` runs with **no framework installed**
- [ ] `curl -i` returns a real 200 for `/api/health` and a real 404 for `/nope`
- [ ] You can say what is in `req` without running it
- [ ] A `POST` with malformed JSON returns **400**, and the server survives
- [ ] You have written at least two middlewares by hand
- [ ] You can explain all **8 steps** of a page load out loud
- [ ] You can explain why `await` does not block, using a diagram
- [ ] You know what CPU-bound work does to a Node server
- [ ] You can say in one sentence what a server is

---

## Check yourself before Chapter 4

1. **What is a server?** One sentence, no jargon.
2. **Walk through what happens when a browser loads a URL.** All 8 steps.
3. **What does `req.url` contain that surprises people?**
4. **Why must you use `curl -i`?**
5. **What is middleware, and what happens if it both responds and calls
   `next()`?**
6. **401 or 403 for a revoked token? What does the client do?**
7. **Why does `await` not block, even though Node is single-threaded?**
8. **What kind of work blocks the event loop, and where does that happen in a
   real app?**
9. **What is Hono, given that you have already written a server?**
10. **Which HTTP code do you return for "route exists but you sent GET"?**

---

**Next: [Chapter 4 — Backend Foundation](./04-backend-foundation.md)** — the
real API. Hono, MongoDB, config that cannot be wrong, logging that cannot leak,
and errors that do not expose internals. Same loop you just built, made
production-safe.