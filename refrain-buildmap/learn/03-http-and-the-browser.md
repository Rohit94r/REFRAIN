# 03 — HTTP, and why extensions can do what a web app cannot

> **Read this after Chapter 7.** By then you will have written a content
> script that reads another site's DOM, and this explains why that is legal and
> why the same code in a web app would be a `SecurityError`.

---

## 1. The request/response model

Every network call is the same shape.

```
CLIENT                                    SERVER
  │  ──▶  GET /profile HTTP/1.1             │
  │        Host: api.refrain.dev            │
  │        Authorization: Bearer <jwt>      │
  │                                        │
  │  ◀──  200 OK                            │
  │        Content-Type: application/json  │
  │        { "name": "..." }               │
```

**The status codes you must know cold:**

| Code | Meaning | Your use |
|---|---|---|
| **200** | OK, and here is the body | Normal success |
| **201** | Created | `POST /sync/push` with new docs |
| **204** | OK, no body | Token refresh returning nothing |
| **301/308** | Permanent / permanent redirect | `refrain.dev` → `www.` |
| **400** | Bad request — malformed | Should never happen; a client bug |
| **401** | **Unauthenticated** — "who are you?" | Token missing or expired |
| **403** | **Forbidden** — "I know who you are, no." | **Token valid but revoked** |
| **404** | Not found | Unknown route |
| **409** | Conflict | Concurrent edit needing a merge |
| **413** | Payload too large | File over 10MB |
| **422** | Unprocessable | Zod validation failed |
| **429** | Too many requests | Rate limited — back off |
| **500** | Server broke | Your bug |

### 401 vs 403 — the bug that causes infinite loops

This is asked in nearly every auth interview and it has a non-obvious answer.

**401 means "I do not know who you are."** The client should get a new token and
retry.

**403 means "I know exactly who you are, and you are not allowed."** Retrying
with the same credentials will never work.

```ts
// ❌ THE BUG
async function refresh() {
  const r = await fetch("/auth/refresh")   // user revoked this device
  // refresh returns 401, correctly
  if (r.status === 401) {
    await refresh()                        // infinite loop. Tab freezes.
  }
}
```

**Why it happens:** self-revocation is real. A user clicks "sign out of this
device" in Chapter 14. That device's refresh token is revoked, so its next
refresh **must** return 401. If the client treats 401 as "refresh and retry," it
retries forever.

**The fix is one line of discipline:**

```ts
// ✅
async function refresh() {
  const r = await fetch("/auth/refresh")
  if (r.status === 403) {
    // Revoked. Do NOT retry. Log out.
    return null
  }
  if (r.status === 401) {
    // Expired, but the refresh token itself is valid. Retry ONCE.
    return refresh()
  }
  return r.json()
}
```

> **Chapter 14 has this bug in its own code on purpose, and Chapter 16's
> contract test catches it.** This is the sort of thing you cannot learn from
> reading — the code looks fine.

---

## 2. The same-origin policy — the whole reason this is an extension

Your instinct for a form-filling tool is a web app. **That is impossible.**

### What the policy says

A page at `refrain.dev` may only read and write **that origin's** DOM and
network. `forms.google.com` is a different origin, so:

```js
// On https://refrain.dev
document.querySelector("#student-name")   // null — wrong document
fetch("https://forms.google.com/…")       // ✗ blocked unless CORS allows it
```

### What counts as "same origin"

**Scheme + host + port.** All three.

| URL | Same as `https://refrain.dev`? |
|---|---|
| `https://refrain.dev/profile` | ✅ yes — path is ignored |
| `https://refrain.dev:8443` | ❌ — port differs |
| `http://refrain.dev` | ❌ — scheme differs |
| `https://www.refrain.dev` | ❌ — host differs |
| `https://evil-refrain.dev` | ❌ — different host |

> **The one that bites in practice:** `http://localhost:5173` and
> `http://localhost:3000` are **different origins** because the ports differ.
> This is why Chapter 17's API sets `CORS_ORIGINS` to include both, and why a
> local dev setup fails with a CORS error when only one is listed.

### CORS — the server's permission slip

CORS does **not** protect your page from attackers. It protects the *server*
from being called by pages it does not trust.

```http
Access-Control-Allow-Origin: https://refrain.dev
Access-Control-Allow-Credentials: true
Access-Control-Allow-Headers: Authorization
```

**Two things people get wrong:**

1. **`Access-Control-Allow-Origin: *` cannot be combined with credentials.**
   If you send cookies or auth headers, the origin must be listed explicitly.
2. **A preflight `OPTIONS` request** is sent automatically before any request
   carrying `Authorization`, `PUT`, or `DELETE`. If your server 404s on
   `OPTIONS`, the real request never fires — and the browser's error message
   mentions neither.

### So how do extensions do it?

A content script does **not** make a network request. **It is injected into the
page's own context**, so `document` *is* the page's document. There is no
cross-origin read happening at all — it is the page's own DOM.

| | Web app | Content script |
|---|---|---|
| Runs in | Its own tab | **The page's tab** |
| `document` refers to | `refrain.dev` | `forms.google.com` |
| Same-origin applies? | Yes, blocks it | **No — it IS the page** |
| Needs host permission | No | **Yes** |

> **The interview answer:** "A web app cannot read another origin's DOM —
> that's the same-origin policy and it is not bypassable. A content script runs
> *inside* the target page's context, so it reads that page's own DOM with no
> cross-origin access involved. What it does need is a host permission, which
> is why our manifest lists specific form domains rather than `<all_urls>` —
> both for user trust and for store review."

---

## 3. Extension messaging

Three contexts, and they cannot touch each other directly.

```
┌─────────────────┐   ┌──────────────────┐   ┌─────────────────┐
│  Content script │   │  Service worker  │   │   Side panel    │
│  (page context) │   │   (background)   │   │  (extension pg) │
└────────┬────────┘   └────────┬─────────┘   └────────┬────────┘
         │                     │                      │
         └──────── chrome.runtime.sendMessage ────────┘
```

**Everything crosses via `sendMessage`**, which is asynchronous and
promise-based:

```ts
// Content script → background
const { schema } = await chrome.runtime.sendMessage({ type: "GET_SCHEMA" })

// Background → content script (needs a tabId)
chrome.tabs.sendMessage(tabId, { type: "FILL", values })
```

### Three gotchas that will cost you hours

**1. Returning a Promise from a `sendMessage` listener requires MV3 syntax.**

```ts
// ❌ MV2. Chrome 2024+ silently never resolves this.
chrome.runtime.onMessage.addListener((msg, sender) => {
  return schema          // a plain value is NOT a response in MV3
})

// ✅ MV3 — mark the listener async, or return true explicitly
chrome.runtime.onMessage.addListener((msg, sender, sendResponse) => {
  sendResponse(schema)
  return true             // keeps the channel open for async
})

// ✅ Or, cleaner, use an async listener
chrome.runtime.onMessage.addListener(async (msg) => {
  return schema
})
```

**2. A failing listener returns `undefined`, which is indistinguishable from
"no response."** Always wrap in try/catch on the caller side, or a single
rejected promise hangs your UI forever.

**3. The service worker dies.** MV3 workers are terminated after ~30 seconds
idle. **An in-memory variable does not survive.** This is why Chapter 6 stores
the derived key in `chrome.storage.session` and re-reads it, and why
`lock()` must be called on every session clear.

> That third one is the most commonly asked: *"Your service worker can die at any
> time. Where do you keep state?"* Answer: `chrome.storage.session` for
> anything that must survive eviction, memory only for a cache you can rebuild.

---

## 4. Content Security Policy

CSP restricts what a page may load. It is the main defence against XSS.

```
script-src 'self'    ← only this origin's scripts. No inline, no eval.
object-src 'self'
frame-ancestors 'none'   ← nobody may iframe me
```

**Why `'unsafe-inline'` and `'unsafe-eval'` are so bad:** an injected
`<script>` tag or an `eval()` call both execute with the page's full privilege.
Allowing either means any XSS becomes arbitrary code execution.

### MV3 is stricter than any web page

```jsonc
{
  "content_security_policy": {
    "extension_pages": "script-src 'self'; object-src 'self'"
  }
}
```

**And MV3 forbids remotely hosted code entirely.** You cannot fetch a
`.js` file from a CDN at runtime.

**This is a hard constraint with a real consequence**, and it caught us during
Chapter 3's build: Tesseract's core is a **WASM module**, which is executable
code. So it cannot be lazily fetched from a CDN — it must be **inside the
extension zip**, which is why the bundle is ~15MB rather than ~3MB.

> **This is a genuine interview insight:** "MV3 bans remote code, and WASM is
> code. So our OCR engine is bundled rather than fetched, which costs us 8MB
> in the zip but is forced to lazy-load via dynamic `import()` — bundled is not
> the same as loaded."

### `frame-ancestors 'none'` and why it matters more here

A clickjacking defence on a normal site: someone iframes you and overlays a fake
button.

**On a review screen, it is a data-theft risk.** Your review screen displays a
student's name, CGPA, and address. An attacker iframes it under a fake "Confirm"
button, the user types a correction, and the keystrokes go to the attacker.

---

## 5. Cookies, `localStorage`, IndexedDB

| | `localStorage` | IndexedDB |
|---|---|---|
| Size | ~5MB | **Hundreds of MB** |
| Values | **Strings only** | Anything, structured-cloneable |
| Async | No, synchronous | Yes |
| Right for | Settings, theme | **Refrain's vault** |

`localStorage` is synchronous, which means reading it **blocks the main
thread**. On the 400ms panel-open budget in Chapter 18, a synchronous read of a
large profile is a measurable regression.

**So: IndexedDB via Dexie.** Async, structured, and it supports transactions
and indexes — which is what Chapter 13's `syncmeta` design needs.

---

## 6. Questions you must be able to answer

1. What three parts make up an origin? Which one surprises people most?
2. **401 vs 403** — why does treating 401 as retryable cause an infinite loop?
3. What does CORS actually protect? (Not what people say.)
4. Why can a content script read another page's DOM when a web app cannot?
5. What host permission does Refrain need, and why not `<all_urls>`?
6. Name the three extension contexts and how they communicate.
7. MV3 `onMessage`: what must a listener return to respond?
8. Why does the extension store its key in `chrome.storage.session` and not a
   module variable?
9. Why is the extension 15MB instead of 3MB? What makes that mandatory?
10. What does `frame-ancestors 'none'` protect against on a review screen?
11. `localStorage` or IndexedDB for a 40MB encrypted vault? Why?

---

*Next: [05 — Encryption](./05-encryption-first-principles.md), or back to the
[fundamentals index](./README.md).*