# Chapter 3 — Your First Server (Very Basic)

> **Day 3 · Goal: you will write a real server by yourself and understand every single line.**
>
> **In this chapter I explain everything in very simple words. No hard words without explaining them first.**
>
> **No framework. No database. Just Node.js and a file called `server.js`.**

---

## Before you start: read this

This chapter is different from the other chapters. The other chapters assume you
already know things. **This one assumes you know nothing.**

That is on purpose. Today you build a server from scratch so that tomorrow, when
you open Chapter 4 and see Hono and MongoDB, you will understand **what those
tools are hiding from you.**

Here is the plan for today:

| Step | What you will do | Time |
|---|---|---|
| 1 | Make a new folder and put a file inside it | 5 min |
| 2 | Write a server that answers "hello" | 10 min |
| 3 | Run it and open it in the browser | 5 min |
| 4 | Add more pages to it | 10 min |
| 5 | Learn what `req` really contains | 10 min |
| 6 | Learn how to read data that a browser sends you | 15 min |
| 7 | Learn about middleware | 20 min |
| 8 | Learn the status codes | 15 min |
| 9 | Understand why `async` exists | 15 min |

**Total: about 2 hours.** Take breaks. Take your time.

---

## Understand this first — Words you need to know

I use these words a lot. Let me explain them **once**, here, in the simplest
way I can.

### Server

A server is just a **program that waits for someone to talk to it.**

That is all. There is nothing magic about it.

Compare it to a shop. The shop is closed and waiting. A customer walks in. The
shopkeeper does something. The customer leaves. Now the shop is waiting again.

- Shop waiting = server waiting
- Customer walks in = someone connects
- Shopkeeper does something = your code runs
- Customer leaves = response is sent

**Your program will do this forever. It will never exit by itself.**

### Request

When someone asks your server something, that ask is called a **request.**

Example: your browser asks `http://localhost:3000/` — that ask is a request.

### Response

What your server sends back is called a **response.**

### HTTP

The language computers use to ask and answer. It is just text with some rules.

### Port

Every program on your computer that wants to talk to the internet needs a
number, so other programs know who to talk to. That number is a **port.**

`http://localhost:3000` — the `3000` is the port.

**Why 3000?** Because 80 and 443 are reserved for real websites. When you are
learning, use 3000 or 8080 so you do not fight with anything else.

### localhost

Means "this computer". When you type `localhost:3000` in your browser, you are
talking to a program running on your own machine. Nobody else can reach it.

---

## Step 1 — Make the folder and the file

This is the step people get stuck on, so let me be very clear.

### 1.1 Open your Terminal

On macOS: press `Command + Space`, type `Terminal`, press Enter.

You should see a black window with a blinking cursor. Good.

### 1.2 Go to your home folder and make a new folder

Copy this and paste it into Terminal, then press Enter:

```bash
cd ~
mkdir refrain-server
cd refrain-server
```

**What each line does:**

- `cd ~` → go to your home folder (your user folder)
- `mkdir refrain-server` → make a new folder. `mkdir` means "make directory"
  (folder). `refrain-server` is the name we are giving it.
- `cd refrain-server` → go **into** that folder

### 1.3 Check you are in the right place

```bash
pwd
```

You should see something like:

```
/Users/rohitjadhav/refrain-server
```

That is your **full address** to this folder. `pwd` means "print working
directory". Very useful — when something goes wrong, this tells you where you
actually are.

### 1.4 Make the file

```bash
touch server.js
```

This creates an empty file called `server.js` in the current folder.

**`.js` means JavaScript.** That is the file type Node.js runs.

### 1.5 Open it in your editor

If you use VS Code (I am assuming you do):

```bash
code server.js
```

If `code` does not work, that means VS Code is not installed. In that case, open
VS Code first, then use **File → Open File** and pick `server.js`.

> **The file is `~/refrain-server/server.js`.** That is the exact location. Full
> path: your home folder → `refrain-server` folder → `server.js` file.
>
> **This folder is temporary.** At the end of today you will delete it. We are
> only using it so you can learn without the monorepo getting in the way.

### 1.6 Make sure Node is working

```bash
node --version
```

You should see something like `v26.10.0`. Any `v20` or higher is fine.

If it says `command not found`, stop and fix Node before continuing.

---

## Step 2 — Write your first server

Open `server.js`, delete everything in it, and paste this **whole thing**:

```js
// This file is a web server.
// A server is just a program that waits for someone to talk to it.

import { createServer } from "node:http"

/*
 * createServer takes one function.
 * That function runs EVERY TIME someone connects.
 * It receives two things:
 *   req = the request  (what the browser asked)
 *   res = the response (how we answer back)
 */

const server = createServer((req, res) => {
  // This prints to the terminal every time someone visits.
  // Good for learning. Remove it later.
  console.log("Someone connected:", req.url)

  // writeHead means "set the status code and the type of content".
  // 200 means "everything is fine".
  // end means "I am finished sending."
  res.writeHead(200, { "Content-Type": "text/plain" })
  res.end("Hello from Refrain")
})

// Now we actually start the server and open the door.
// Without this line nothing happens. No error, nothing. Just silence.
server.listen(3000, () => {
  console.log("Server is running. Open http://localhost:3000")
})
```

### Now let me explain every single line

**Line 1–2** — a comment. The `//` means "this is a note for humans, the
computer ignores it." Use comments to explain things. Write more of them.

**`import { createServer } from "node:http"`**

This line says: "I want the thing called `createServer` from Node's built-in
`http` module."

- `createServer` is the function that makes a server
- `node:http` is the built-in tool that has it
- You do **not** need to install anything. It comes with Node.

Note the `"` quotes — always use quotes around a module name like this.

**`const server = createServer((req, res) => { ... })`**

- `const` means "I am making a variable that will not change"
- `server` is the name we give it
- `=>` is an arrow function. It means "this function is so short we skip the
  word `function` and the word `return`"
- `(req, res)` are the two things Node gives us: the request and the response
- `{ ... }` is the body of the function — the actual work

**`req` = the request.** What the browser sent you. A question.

**`res` = the response.** What you send back. The answer.

**`console.log("Someone connected:", req.url)`**

Prints something in your terminal so you can see what happened. Good for
learning.

**`res.writeHead(200, { "Content-Type": "text/plain" })`**

- `writeHead` = set the label on the answer before you write the answer
- `200` = the status code. It means "fine, here is what you asked for"
- `"Content-Type"` tells the browser **what kind** of thing is coming
- `"text/plain"` means "just plain words, no fancy formatting"

**`res.end("Hello from Refrain")`**

Finishes the answer and sends these exact words to the browser.

> **Rule to remember: you must always call `res.end()`.** If you forget, the
> browser will keep waiting forever and the page will just spin. This is one of
> the most common mistakes. If a page never loads, check for a missing `res.end()`.

**`server.listen(3000, () => { ... })`**

- `listen` = start the server and wait for connections
- `3000` = the port number
- The function after it runs **one time**, when the server starts. It does not
  run for every visitor. It just tells you "we are open".

---

## Step 3 — Run it and look at it

### 3.1 Start the server

In Terminal:

```bash
node server.js
```

You should see:

```
Server is running. Open http://localhost:3000
```

**Your Terminal is now stuck.** That is correct. The server is waiting. Do not
close this window.

### 3.2 Open it in the browser

Open Chrome. Type this in the address bar:

```
http://localhost:3000
```

You should see, in big plain letters:

```
Hello from Refrain
```

**If that worked, you just wrote a web server.** That is the whole thing.

### 3.3 Look at your Terminal

Every time you refresh the browser, a new line appears:

```
Someone connected: /
Someone connected: /
Someone connected: /
```

That is your server hearing visitors. It is listening.

### 3.4 Stopping the server

In the Terminal, press `Control + C`.

That kills the program. Now the browser will show an error. That is fine — the
server is stopped.

**To start it again:** `node server.js`

---

## Step 4 — Add more pages

A server that only says "Hello" is not very useful. Let's make it answer
differently depending on what the browser asks for.

Replace everything in `server.js` with this:

```js
// A server that answers differently for different addresses.

import { createServer } from "node:http"

const server = createServer((req, res) => {
  console.log(`${req.method} ${req.url}`)

  // ── Page 1 ────────────────────────────────────────────────
  // If the address is exactly /api/health, say "I am healthy".
  if (req.url === "/api/health") {
    res.writeHead(200, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ ok: true, time: new Date().toISOString() }))
    return // stop here. Do not check the other ifs.
  }

  // ── Page 2 ────────────────────────────────────────────────
  // If the address is / , say hello with HTML.
  if (req.url === "/") {
    res.writeHead(200, { "Content-Type": "text/html; charset=utf-8" })
    res.end("<h1>Refrain</h1><p>Answer once. Fill everywhere.</p>")
    return
  }

  // ── Page 3 ────────────────────────────────────────────────
  // Anything else does not exist.
  // 404 means "I looked, but there is nothing here".
  res.writeHead(404, { "Content-Type": "application/json" })
  res.end(JSON.stringify({ error: "not_found" }))
})

server.listen(3000, () => {
  console.log("Server is running. Open http://localhost:3000")
})
```

### What is new here

**`if (req.url === "/api/health")`**

"if what the browser asked equals `/api/health`" — then do this.

**`===` is "is exactly equal to".** Always use three `=` signs. `=` means
"put a value into a variable" (different job). `==` means "is equal, sort of"
(avoid it, it has confusing rules).

**`return`**

Stop this function right now. Without `return`, the code keeps going and you
might send **two** answers. The browser then gets confused and may hang.

**`JSON.stringify({...})`**

Turns a JavaScript object into text so it can travel over the internet.

`{ ok: true }` is a JavaScript object. On the wire it becomes the text
`{"ok":true}`.

**`new Date().toISOString()`**

The current date and time, in a standard text format.

**`charset=utf-8`** — how to read the words. `utf-8` can write every language,
including Hindi. Always include it for text.

### Test all three

Start the server again: `node server.js`

Then open each of these in your browser:

| Type this | You should see |
|---|---|
| `http://localhost:3000` | Big **Refrain** heading |
| `http://localhost:3000/api/health` | `{"ok":true,"time":"2026-..."}` |
| `http://localhost:3000/nope` | `{"error":"not_found"}` |

All three worked? You just wrote **routing**. That is the whole idea.

### Now try this — it breaks on purpose

Visit `http://localhost:3000/api/health?debug=1`

**Nothing changes. You still get the answer.** But look at your Terminal — it
printed the full address **including** `?debug=1`.

So `req.url` was `/api/health?debug=1`, which is **not equal** to
`/api/health`. So the first `if` failed. It fell through to the last one, which
gave you 404.

**This is a real bug in every hand-written server ever made.** Chapter 4's Hono
does the splitting for you and never has this problem. But now **you** know why.

> **The fix, when you need it:** split the query string off first.
>
```js
// Split the query string off. "/api/health?debug=1" becomes "/api/health".
const [path] = req.url.split("?")
if (path === "/api/health") {
  // this now matches, even with ?debug=1 attached
}
```
>
> You do not need this today. Just know it exists.

---

## Step 5 — Look at what actually arrives

I want you to *see* a request. Stop guessing.

Replace everything in `server.js` with:

```js
// A server that just shows you what the browser sent.

import { createServer } from "node:http"

const server = createServer((req, res) => {
  // This prints everything the browser sent us.
  console.log("---- WHAT I RECEIVED ----")
  console.log("method: ", req.method)
  console.log("url:    ", req.url)
  console.log("headers:", req.headers)
  console.log("-------------------------")

  res.writeHead(200, { "Content-Type": "text/plain" })
  res.end("look at your terminal")
})

server.listen(3000, () => console.log("running"))
```

Run it. Open `http://localhost:3000/some/path?a=1`. Look at the Terminal.

### Three surprises waiting for you

**1. `req.url` has the `?` part stuck on it.**

```
url:     /some/path?a=1
```

The part after `?` is called the **query string**. Node gives you everything as
one string. You have to split it yourself.

**2. Headers are always lowercase.**

You will see `content-type`, never `Content-Type`. If you look for
`Content-Type` you will get `undefined`.

**3. `req.method` is uppercase.** `GET`, `POST`, `PUT`, `DELETE`. Always
uppercase. Comparing with lowercase `"get"` fails.

### The real lesson

**A request is just a plain object.** No magic. You can `console.log` it and read
every property. That is worth knowing, because people imagine the browser sends
something mysterious.

---

## Step 6 — Receiving data that the browser sends

So far the browser has only *asked* for things. What if it sends **data**?

Example: a form where the user types their name, and the browser sends the name
to your server.

### 6.1 First, the problem

A `POST` body arrives **in pieces**. It might arrive in one piece, or a hundred,
or all at once — you do not know until it has finished.

So you cannot read it in one go. You have to collect pieces until it says "I am
done".

**This is why it must be `async`.** If you tried to read it synchronously, the
data might not have arrived yet.

### 6.2 The code

Replace everything in `server.js` with:

```js
// A server that receives data.

import { createServer } from "node:http"

const server = createServer((req, res) => {
  // If this is not a POST, tell them and stop.
  if (req.method !== "POST") {
    // 405 means "this page exists, but not for the way you asked".
    res.writeHead(405, { Allow: "POST" })
    res.end()
    return
  }

  // An empty string to collect pieces into.
  let body = ""

  // This runs MANY times. Each time we get one piece of data.
  req.on("data", (piece) => {
    body = body + piece
  })

  // This runs ONCE, at the end, when all the pieces have arrived.
  req.on("end", () => {
    // Now it is finally safe to turn the text into an object.
    let data
    try {
      data = JSON.parse(body)
    } catch {
      // The data was broken. That is the USER'S fault, not ours.
      // 400 means "what you sent was not understandable".
      res.writeHead(400, { "Content-Type": "application/json" })
      res.end(JSON.stringify({ error: "invalid_json" }))
      return
    }

    // Everything worked.
    res.writeHead(201, { "Content-Type": "application/json" })
    res.end(JSON.stringify({ id: "doc_1", ...data }))
  })
})

server.listen(3000, () => console.log("running"))
```

### 6.3 What is new here

**`let body = ""`**

`let` is like `const`, but the value **can change** later. We start with an
empty string and add pieces to it.

**`req.on("data", ...)`**

"Every time data arrives, run this function." It runs many times.

**`req.on("end", ...)`**

"After all the data has arrived, run this." It runs **one** time.

**`try { } catch { }`**

"Try this. If it throws an error, run that instead."

Without `try/catch`, one broken request would **crash your whole server**. That
is a real attack: someone sends garbage, your server dies, nobody can use it.

**`...data` (the three dots)**

Called spread. It copies all the properties of `data` into the new object. So if
the user sent `{"name":"refrain"}`, you send back
`{"id":"doc_1","name":"refrain"}`.

**201 instead of 200**

201 means "I made something new". 200 means "here is what you asked for". Small
detail, but it is the correct code.

### 6.4 Test it

Start the server: `node server.js`

Now open a **new Terminal tab** (`Command + T`) so you can type commands while
the server runs.

**Test 1 — correct data:**

```bash
curl -i -X POST http://localhost:3000/api/docs \
  -H "Content-Type: application/json" \
  -d '{"name":"refrain","encrypted":true}'
```

You should see:

```
HTTP/1.1 201 Created
Content-Type: application/json

{"id":"doc_1","name":"refrain","encrypted":true}
```

**Test 2 — broken data:**

```bash
curl -i -X POST http://localhost:3000/api/docs \
  -H "Content-Type: application/json" \
  -d '{bad json'
```

You should see:

```
HTTP/1.1 400 Bad Request

{"error":"invalid_json"}
```

**Now try test 1 again. Does it still work?**

**Yes? Good. Your server did not crash.** That is the whole point of `try/catch`.

### 6.5 What `curl` is

`curl` is a command-line tool that acts like a browser. It is perfect for
testing servers because it shows you everything.

- `curl` = talk to the server
- `-i` = **also show me the status code and headers**
- `-X POST` = use the POST method
- `-H "..."` = add this header (like Content-Type)
- `-d '...'` = send this data

**Always use `-i`.** Without it, you see only the answer text and cannot tell a
200 from a 404.

> **Tip:** the `\` at the end of a line lets you continue the command on the next
> line. It is not part of the command. You can also write it all on one line.

### 6.6 A danger you have not solved yet

```js
req.on("data", (piece) => {
  body = body + piece   // ⚠️ no limit!
})
```

**What if someone sends 10GB?** Your server will try to hold it all in memory
and **crash**. The whole server goes down.

Every real server puts a limit here. Ours will be 12MB, because the biggest
thing a user uploads is a 10MB document.

You do not need to solve this today. **But now you know the problem exists**, and
you will fix it properly in Chapter 4.

---

## Step 7 — Middleware

Right now your one big function does everything: logging, reading the body,
and answering. That gets messy fast.

**Middleware is just a function that runs before your main function.**

### 7.1 The idea

Think of airport security:

1. You arrive → security checks your bag (middleware)
2. If your bag is bad → you are sent home, you never reach the gate
3. If your bag is fine → you continue to the gate (your actual handler)

Middleware can either **stop the request** or **let it continue.**

### 7.2 The code

Replace everything in `server.js`:

```js
// Same server, but organised better using middleware.

import { createServer } from "node:http"

// ── Middleware 1: log every request ─────────────────────────────
function logger(req, res, next) {
  const startTime = Date.now()

  // "finish" runs after we have sent the full answer.
  res.on("finish", () => {
    const ms = Date.now() - startTime
    console.log(`${req.method} ${req.url} -> ${res.statusCode} in ${ms}ms`)
  })

  next() // let the request continue
}

// ── Middleware 2: turn the body text into a real object ─────────
function readJsonBody(req, res, next) {
  let body = ""

  req.on("data", (piece) => {
    body = body + piece
  })

  req.on("end", () => {
    try {
      req.body = body ? JSON.parse(body) : {}
      next() // it worked, continue
    } catch {
      // it did not work, stop here. Do NOT call next().
      res.writeHead(400, { "Content-Type": "application/json" })
      res.end(JSON.stringify({ error: "invalid_json" }))
    }
  })
}

// ── The main server ─────────────────────────────────────────────
const server = createServer((req, res) => {
  // Run logger. When it is done, run readJsonBody.
  // When that is done, run the real handler below.
  logger(req, res, () => {
    readJsonBody(req, res, () => {
      // This is the real work. Notice how clean it is now.
      res.writeHead(200, { "Content-Type": "application/json" })
      res.end(JSON.stringify({ youSentMe: req.body }))
    })
  })
})

server.listen(3000, () => console.log("running"))
```

### 7.3 What is `next`

`next` means "I am done, let the next thing happen."

Our `createServer` handler takes only `(req, res)`. The framework **passes a
third function called `next` into our middlewares.**

So every middleware has three things: `req`, `res`, and `next`.

- Do your job
- Call `next()` to continue
- **Or** send a response and do NOT call `next()`

### The one rule

**A middleware either sends an answer, or calls `next()`. Never both.**

If you send an answer AND call `next()`, your code sends a **second** answer.
The browser gets two answers, gets confused, and often hangs.

Choosing to stop (sending an answer without calling `next`) is **allowed and
correct** — that is exactly how `readJsonBody` rejects bad data.

### 7.4 Test it

```bash
curl -i -X POST http://localhost:3000/anything \
  -H "Content-Type: application/json" \
  -d '{"hello":"world"}'
```

```bash
curl -i -X POST http://localhost:3000/anything \
  -H "Content-Type: application/json" \
  -d '{oops'
```

Now look at your **Terminal**. You should see neat log lines:

```
POST /anything -> 200 in 2ms
POST /anything -> 400 in 1ms
```

Those are the two middlewares doing their jobs.

### 7.5 Why this matters

**Every backend framework is built on this exact idea.**

| Framework | What middleware is called |
|---|---|
| Express | `app.use(...)` |
| Hono | `app.use(...)` |
| FastAPI | "dependencies" |

Same idea, different name. **So once you understand this, you already know how
every backend framework works.** Chapter 4 introduces Hono and this will feel
familiar.

---

## Step 8 — Status codes

These numbers are how your server tells the browser what happened.

**You must memorise these.** They come up in every interview and every debugging
session.

| Code | Name | When to use it |
|---|---|---|
| **200** | OK | It worked. Here is the answer. |
| **201** | Created | You made something new. |
| **204** | No Content | It worked, and I am deliberately sending nothing back. |
| **301** | Moved | This page has moved permanently. |
| **400** | Bad Request | What you sent was broken. **Their fault.** |
| **401** | Unauthorized | I do not know who you are. |
| **403** | Forbidden | I know who you are. **No.** |
| **404** | Not Found | I looked. There is nothing here. |
| **405** | Method Not Allowed | The page exists, but not for GET/POST like that. |
| **409** | Conflict | Two people changed the same thing. |
| **413** | Too Large | What you sent is too big. |
| **429** | Too Many Requests | You are asking too fast. Slow down. |
| **500** | Server Error | **My fault.** Something broke. |

### 401 versus 403 — this one is asked a lot

**401 = "I do not know who you are."**
The client should get new credentials and try again.

**403 = "I know exactly who you are, and you are not allowed."**
Trying again with the same credentials will **never** work.

### The infinite loop nobody warns you about

```js
// ❌ WRONG — this hangs the browser forever
async function refreshToken() {
  const response = await fetch("/auth/refresh")

  if (response.status === 401) {
    await refreshToken()   // calls itself forever!
  }
}
```

**Why this really happens in Refrain:**

Chapter 6 lets a user sign out of one device. That kills that device's refresh
token. So next time that device tries to refresh, it correctly gets **401**.

This client sees 401 and refreshes again. Forever. The tab freezes and you have
to close the browser.

**The fix:**

```js
async function refreshToken() {
  const response = await fetch("/auth/refresh")

  if (response.status === 403) return null            // signed out. Stop.
  if (response.status === 401) return refreshToken()  // expired. Try once more.
  return response.json()
}
```

> **Chapter 6 has this exact bug on purpose.** Chapter 7 writes a test that
> catches it. **You cannot learn this by reading about it. You have to hit it.**

### The rule for errors

**Never send 200 with an error inside.**

```js
// ❌ WRONG
res.writeHead(200)
res.end(JSON.stringify({ error: "not found" }))

// ✅ CORRECT
res.writeHead(404)
res.end(JSON.stringify({ error: "not found" }))
```

Both send the same words. But the **number** tells every computer in the chain
what happened. Caches store it, monitoring counts it, browsers handle retries
by it. Send 200 with an error and all of those break silently.

---

## Step 9 — Why `async` exists

Your server works for one user. What about 5,000 at once?

### 9.1 The problem

```js
// ❌ This is the problem we are solving
const server = createServer(async (req, res) => {
  const data = await someSlowDatabaseCall()  // takes 500ms
  res.end(data)
})
```

If Node **waited** for that 500ms before looking at other requests, then user
A's slow request would block users B, C, and D.

### 9.2 The answer

**`await` does not wait. It pauses and lets go.**

When Node hits `await`, it:

1. Stops this function
2. Goes and looks at other requests
3. Comes back when the database answers

```
Request A         Request B         Request C
   |                 |                 |
   |--await----->    |--await---->    |--await---->
   |                 |                 |
(doing other      (doing other      (doing other
  things)           things)           things)
   |                 |                 |
   |<----done--------|<----done-------|<----done-----
   v
 answers
```

**All three requests are moving forward at the same time, on one thread of
JavaScript.** They are not taking turns — they are all waiting on something.

### 9.3 When this breaks

`await` is great for **waiting**. It is terrible for **working**.

```js
// ✅ Fine — this runs on a special thread
const hash = await crypto.subtle.digest("SHA-512", data)

// ❌ Bad — this runs on the main thread and blocks EVERYONE
const sorted = hugeArray.sort((a, b) => a - b)
```

If one user triggers a big sort, **every other user's request stops** until it
finishes. Four seconds of nothing.

**This is why Refrain does OCR in a "Worker"** in the browser — so the heavy
work happens somewhere else.

### 9.4 The interview answer

> "Node runs JavaScript on one thread, and uses a pool of extra threads for
> things like file reading and WebCrypto. So it is very good at handling
> thousands of requests that are **waiting** for something, and bad at work
> that **takes time to compute**. `async` lets many requests be in flight at
> once, but a long calculation still blocks everyone."

That answer is worth marks on its own. Learn it.

---

## Step 10 — What you built today

With **no framework at all**, you wrote:

- A working web server
- Three pages, chosen by address (routing)
- Correct 404 and 405 and 400 codes
- Reading data sent by the browser
- Two middlewares
- Logging with timing

**Chapters 4 to 8 are this same thing, plus a database.**

| What you wrote by hand | What a framework gives you |
|---|---|
| `if (req.url === "/api/health")` | `app.get("/api/health", handler)` |
| `req.on("data", ...)` | `await c.req.json()` |
| Your own middleware | `app.use(logger)` |
| `createServer` + `listen` | **The same thing.** Still Node. |

**Hono is not a server. It is a shortcut for routing.** Underneath, it is
exactly the `createServer` loop you wrote today.

---

## Step 11 — Clean up

You are done. Delete the scratch project:

```bash
cd ~
rm -rf refrain-server
```

**Do not skip this.** Chapter 4 builds the real server inside your project at a
different address. If you leave this one behind, you will one day run the wrong
file and spend an hour confused about why your changes are not showing up.

---

## Your 60/40 split for this chapter (all of it yours)

There is nothing to delegate in this chapter. There is no framework yet.

| # | Task | Who |
|---|---|---|
| 1 | Make the folder and `server.js`. Open it in your editor | **You** |
| 2 | Write Step 2's server **by typing it**, then run it | **You** |
| 3 | Open it in the browser. Refresh it 3 times. Watch the Terminal | **You** |
| 4 | Write Step 4's server. Test all three addresses | **You** |
| 5 | Run Step 5 and **read every property of `req`** | **You** |
| 6 | Run Step 6. Test good data AND broken data | **You** |
| 7 | Run Step 7. Confirm your server survives broken data | **You** |
| 8 | Memorise the status code table into `NOTES.md` | **You** |
| 9 | **Say out loud** what a server is, in one sentence | **You** |
| 10 | **Say out loud** why `await` does not block | **You** |

---

## Gotchas in this chapter — common mistakes

**The browser spins forever and never loads.**
You forgot `res.end()`. Every path must end the response.

**`ReferenceError: createServer is not defined`.**
You forgot the `import` line at the top, or you deleted it while editing.

**`EADDRINUSE: address already in use :::3000`.**
Something else is using port 3000. Either you already started the server and
forgot, or another program wants it. Change 3000 to 4000 in both `listen()` and
your browser address.

**Changes are not showing up.**
You did not save the file. Press `Command + S`. Or you are running an old copy
from the wrong folder — run `pwd` and check.

**`Cannot use import statement outside a module`.**
Your `package.json` is missing `"type": "module"`. Add it, or rename the file to
`server.mjs`.

**Everything looks fine but the Terminal shows nothing.**
You never called `server.listen(...)`. Without that line the server is built but
never starts, and there is **no error message**. This is the most confusing one.

---

## Verify before moving on — check this

- [ ] `server.js` runs with **nothing installed** except Node
- [ ] `http://localhost:3000` shows a heading
- [ ] `/api/health` gives JSON with `"ok":true`
- [ ] `/nope` gives a **404**
- [ ] You can say what is inside `req` without running anything
- [ ] Bad JSON gives **400** and your server keeps working
- [ ] You wrote two middlewares yourself
- [ ] You can say all 13 status codes from memory (or nearly)
- [ ] You can explain in one sentence what a server is
- [ ] You can explain why `await` does not block

---

## Check yourself before Chapter 4

Answer these **out loud**. If you cannot, go back and read that part again.

1. **What is a server?** One sentence. No technical words.
2. **What is a port?** Why do we use 3000 and not 80?
3. **What does `localhost:3000` mean?**
4. **What are `req` and `res`?** Which one is the question?
5. **What does `res.writeHead(200, ...)` do?**
6. **Why must you always call `res.end()`?**
7. **What is the problem if you forget `return` after sending an answer?**
8. **Why did `?debug=1` break our check?**
9. **Are headers `Content-Type` or `content-type`?**
10. **Why must reading a POST body be asynchronous?**
11. **What does `try` / `catch` protect us from?**
12. **What is middleware? In your own words.**
13. **What does `next()` do?**
14. **Why must a middleware never both answer AND call `next()`?**
15. **400 versus 404 — what is the difference?**
16. **401 versus 403 — what is the difference?**
17. **Why does retrying a 401 cause an infinite loop?**
18. **Why is it wrong to send an error inside a 200?**
19. **Does `await` block? Then what does it do?**
20. **What kind of work blocks the whole server, and where does that happen in
    a real app?**

---

**Next: [Chapter 4 — Backend Foundation](./04-backend-foundation.md)** — the
real server, inside your project. Hono, MongoDB, configuration that cannot be
wrong, logging that cannot leak your users' data, and errors that do not show
internals.

Everything in that chapter is what you built today, made safe.