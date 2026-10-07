# Chapter 4 — Backend Foundation

> **Day 4 · Goal: an API running locally against MongoDB, with §11 rewritten
> to tell the truth.**
>
> You asked for MongoDB for the long run. It is the right call for the data shape. It also
> breaks one of your strongest marketing lines. This chapter fixes that honestly instead of
> quietly.

---

## Before you start Phase 2

**You already have a shippable product.** Chapters 1–11 — your fifteen days — produced a
working extension that fills
forms, with an encrypted vault, document extraction, and a tracker — and **no server at all.**

That matters for two reasons:

**One: nothing in Phase 2 is required to ship.** Chapter 20 submits v1.0 without any of it. The
backend is a bet that demand for cross-device sync exists, and it is deliberately built *after*
the thing people might want sync for. If you never get users, the backend cost you nine days
and taught you something you will use again.

**Two: the frontend is now a constraint, not a starting point.** Everything in Chapters 12–18
has to fit under what Chapters 1–11 already committed to. `packages/fields` is published across
a boundary you cannot recall. The vault's key hierarchy was designed before `k_auth` existed —
Chapter 6 has to fit inside it, not redesign it. **Read Chapters 6 and 13's diagrams before you
touch this code.** The cryptography is already decided; you are building the server around it,
not the other way around.

> **The uncomfortable part starts here.** `refrain.md` §11 currently promises *"no server,
> nothing leaves your device."* That is no longer true, and a promise that is no longer true is
> worse than no promise. Step 7 of this chapter has you rewrite §11 yourself. **Do not footnote
> it. Rewrite it.** The replacement positioning — *"works with no account and no network; sync is
> optional and encrypted with a key we never receive"* — is not a downgrade. It is a stronger
> claim, because it is one you can keep.

---

## Understand this first

### What a backend actually buys you

Let me be precise, because "we need a backend" is not a reason.

| What sync gives you | Why it matters |
|---|---|
| Change laptop, keep the profile | A real user need, not a nice-to-have |
| Two devices, one vault | Same |
| Recovery when a laptop dies | **The strongest argument** — right now a lost laptop means a lost profile |
| A referral loop | Nothing. Ignore this one |

Everything else a backend gives you — analytics, a recommendation model, a shared vault for your
team — is either a §11 violation or a Phase 6 problem that needs a compliance story you do not
have yet.

**So the backend is optional infrastructure for optional sync, and it must be removable.** If
you delete the whole `apps/api` folder, the product still works. That is a design constraint,
not a hope.

### Why the server is also TypeScript

This comes up immediately, so answer it once, properly. You are not choosing "Node vs Python"
for the project — **the extension and the web app are already JavaScript because MV3 has no
other option.** The only real question is whether the one server deserves its own language.

It does not, and the reason is one package:

```
              packages/fields/
              FormField · ProfileFact · SyncEntry · ErrorCode
                        ▲                    ▲
             apps/extension            apps/api
```

`apps/api` imports **the same Zod schemas the extension builds its payloads from.** A shape
change fails CI once, in one place, because there is only one shape to change. Move the server
to Python and you own two definitions of the sync protocol with nothing comparing them — and
Chapter 8's three-way merge is exactly the code that fails *silently* when the two disagree.

> **Python's best argument here is the one that does not apply.** It has the best ML ecosystem,
> and the strongest OCR story. But your inference is on-device (Chapter 14) and your OCR is
> in-browser (Chapter 17) — because the marksheet must never be uploaded. **Both of Python's
> best features require the server to see data it is forbidden to see.** Choosing it for those
> would be choosing a language for capabilities you have deliberately ruled out.
>
> **The full comparison, including where Node genuinely loses** (native module builds for
> Argon2id, event-loop blocking, no parallelism) is in
> [`TECH-STACK.md` → Why Node, and not Python or a Next.js backend](./TECH-STACK.md).
> Read it before you defend the choice in a pull request.

### The cost, stated plainly

§11 currently says *"No account. No server. Nothing leaves your device."* All three become
false the moment you ship a backend. Not approximately false. **False, in writing, on a page
you published.**

That matters more than the revenue. §11 is the product. The moment your privacy page and your
network tab disagree, you have destroyed the exact trust you built.

> **I am not telling you not to build the backend.** You asked for it, it is the right long-run
> choice, and cross-device recovery is a genuine user need. I am telling you that the fix is a
> rewrite of §11, not a footnote.

### The rewrite

| Current §11 claim | Becomes |
|---|---|
| "No account. No server." | "No account required. Works entirely offline. Optional sync if you want it." |
| "Nothing leaves your device." | "Everything stays on your device. If you turn on sync, it is encrypted with a key the server never receives." |
| "We have no server." | "We operate a sync server that stores encrypted blobs. It cannot read them. We designed it so it cannot." |

Note the third one. **Do not pretend the backend does not exist.** A reviewer who finds your
`api.refrain.dev` DNS record knows you have a server, and a privacy page that pretends
otherwise is worse than one that admits it. The strength is in *why it cannot read the data*,
not in the absence of a server.

**New one-liner for §11:** *"Refrain works with no account and no network. Sync is optional, and
encrypted with a key we never receive."*

### The constraint that shapes the entire backend

> **A server that cannot read your data cannot query your data.**

If you store opaque ciphertext, you have thrown away every index, every aggregate, and
server-side search that MongoDB exists for. Your `$text` index cannot index AES-GCM output. It
is random bytes.

That is not a small cost. It is the real cost of end-to-end encryption, and pretending
otherwise is how you end up with a backend that quietly logs plaintext because "we needed to
search it."

**Which is why the schema in Chapter 5 is hybrid** — opaque blobs for content, plus a strictly
minimal metadata index so sync can work. And it is why search happens in your browser, not on
your server.

---

## Step 1 — Add the app

```bash
mkdir -p apps/api/src && cd apps/api && pnpm init
```

```jsonc
// apps/api/package.json
{
  "name": "@refrain/api",
  "private": true,
  "type": "module",
  "scripts": {
    "dev": "tsx watch src/server.ts",
    "build": "tsc -p tsconfig.json",
    "start": "node dist/server.js",
    "typecheck": "tsc --noEmit",
    "test": "vitest run",
    "seed": "tsx src/db/seed.ts"
  },
  "dependencies": {
    "@refrain/fields": "workspace:*",
    "hono": "^4.6.0",
    "mongoose": "^8.9.0",
    "zod": "^4.0.0",
    "pino": "^9.5.0",
    "argon2": "^0.41.0",
    "cors": "^2.8.5",
    "rate-limiter-flexible": "^5.0.0"
  },
  "devDependencies": {
    "tsx": "^4.19.0",
    "typescript": "^5.9.0",
    "vitest": "^3.0.0",
    "mongodb-memory-server": "^10.1.0"
  }
}
```

Add the workspace dependency rules that matter:

```ts
// ❌ The API must never import UI code.
import { something } from "@refrain/ui"      // never

// ✅ It shares types and schemas only.
import { FormSchema } from "@refrain/fields" // fine
```

**`@refrain/fields` is the only shared package the API may import.** That package has zero
dependencies and no DOM. The moment the API can import UI code, it can start reading field
labels server-side, and §11 is over.

### Layout

```
apps/api/src/
├── server.ts          ← entry: bind, listen, graceful shutdown
├── env.ts             ← Zod-validated config. Loads first.
├── app.ts             ← the Hono app. Routers mounted here.
├── db/
│   ├── connect.ts     ← MongoClient, pooling, lifecycle
│   ├── models/        ← Mongoose schemas (Chapter 5)
│   └── seed.ts
├── middleware/
│   ├── logger.ts      ← pino, with PII scrubbing
│   ├── auth.ts        ← Chapter 6
│   ├── rateLimit.ts
│   └── errors.ts
├── routes/
│   ├── health.ts
│   ├── auth.ts
│   ├── sync.ts
│   └── account.ts
└── lib/
    ├── crypto.ts      ← server-side hashing only
    └── errors.ts      ← AppError, error codes
```

> **`env.ts` is imported before anything else, and it throws on a missing variable.** Every
> other file assumes `env.MONGODB_URI` is a validated string. Fail at boot, not at 2am on the
> first request. A backend that reads `process.env` in twelve files is a backend with twelve
> chances to get the name wrong.

---

## Step 2 — Config that cannot be wrong

```ts
// apps/api/src/env.ts
import { z } from "zod"

const schema = z.object({
  NODE_ENV: z.enum(["development", "test", "production"]).default("development"),

  MONGODB_URI: z.string().url(),
  MONGODB_DB: z.string().default("refrain"),

  PORT: z.coerce.number().int().positive().default(8787),

  CORS_ORIGINS: z.string()
    .transform((s) => s.split(",").map((o) => o.trim()).filter(Boolean))
    .refine((o) => o.length > 0, "CORS_ORIGINS is required"),

  JWT_SECRET: z.string().min(32),
  /** Public key for clients to verify access tokens. Not secret. */
  JWT_PUBLIC_KEY: z.string().min(32),

  RATE_LIMIT_RPM: z.coerce.number().default(60),

  LOG_LEVEL: z.enum(["debug", "info", "warn", "error"]).default("info"),
})

export type Env = z.infer<typeof schema>

const parsed = schema.safeParse(process.env)

if (!parsed.success) {
  // Fail loudly and specifically. A silent default is a prod incident.
  console.error("Invalid environment:\n", parsed.error.flatten().fieldErrors)
  throw new Error("Invalid environment configuration")
}

export const env: Env = parsed.data
```

```bash
# apps/api/.env.example  — commit this. Never commit .env.
NODE_ENV=development
MONGODB_URI=mongodb://localhost:27017
MONGODB_DB=refrain
PORT=8787
CORS_ORIGINS=http://localhost:5173,chrome-extension://abcdefghijklmnop
JWT_SECRET=generate-with-openssl-rand-base64-48
JWT_PUBLIC_KEY=generate-with-openssl-rsa-pubout
RATE_LIMIT_RPM=60
LOG_LEVEL=debug
```

> **The CORS origin list includes `chrome-extension://<your-id>`.** Extension pages have their own
> origin scheme and a browser extension ID. If you forget it, the side panel cannot talk to the
> API and the error is an opaque CORS failure with no useful message. This bites everyone once.

---

## Step 3 — MongoDB locally

```bash
docker run -d --name refrain-mongo \
  -p 27017:27017 \
  -e MONGO_INITDB_DATABASE=refrain \
  -v refrain-mongo-data:/data/db \
  mongo:8
```

> **Always mount a named volume.** Without `-v`, `docker rm` takes your local data with it, and
> the first time you run `docker system prune` you lose a week of test fixtures. This is
> unglamorous and it will save you.

Or use Atlas for everything, including development:

```bash
# A free M0 tier, with a separate database per branch.
```

> **Use a separate Atlas database per branch.** `refrain_dev_$GITHUB_BRANCH`. Sharing one
> database across two feature branches is how you end up with your test run writing fake users
> into your dev data, and how CI wipes your local state. This is a five-character change that
> prevents a class of outage.

---

## Step 4 — Connection and pooling

```ts
// apps/api/src/db/connect.ts
import mongoose from "mongoose"
import { env } from "../env"

let connected = false

export async function connectDb(): Promise<void> {
  if (connected) return

  mongoose.set("strictQuery", true)          // ignore fields not in your schema
  mongoose.set("sanitizeFilter", true)       // reject { $ne: null } injection

  await mongoose.connect(env.MONGODB_URI, {
    dbName: env.MONGODB_DB,
    maxPoolSize: 10,          // ← the pooling knob. See below.
    minPoolSize: 2,
    serverSelectionTimeoutMS: 5000,   // fail fast, not hang for 30s
    retryWrites: true,
  })

  connected = true

  mongoose.connection.on("error", (e) => console.error("Mongo error", e))
  mongoose.connection.on("disconnected", () => { connected = false })
}

export async function disconnectDb(): Promise<void> {
  if (!connected) return
  await mongoose.disconnect()
  connected = false
}
```

### Why `maxPoolSize: 10`

MongoDB's default is 100 connections per client process. Atlas's limit is roughly **10,000
per cluster.** That sounds like headroom until you scale to three app instances, four workers
each, and a migration job running concurrently:

```
3 instances × 4 workers × 100 default = 1,200   ← fine
... × background jobs × a traffic spike        ← not fine
```

Set it explicitly and low. Ten is plenty for an API where every query is a point read or a
small write, and it makes you immune to the classic
`MongoServerSelectionError: Server selection timed out after 30000ms`.

> **`sanitizeFilter: true` is not optional.** Without it, a request body of
> `{ email: { $ne: null } }` against a login route matches the **first user in the collection**.
> That is not a theoretical injection — it is a full account enumeration in one JSON field.
> Turn this on and never turn it off.

---

## Step 5 — The logger that cannot leak

**This is the most important file in the backend.** Your logs are the one place plaintext
personal data will touch a server, and servers log.

```ts
// apps/api/src/middleware/logger.ts
import pino from "pino"
import { env } from "../env"

/**
 * Fields that must NEVER appear in a log line.
 *
 * This is a denylist, which is weaker than an allowlist — but an
 * allowlist silently breaks when a new dependency adds a field you
 * did not think about. Deny the known-bad loudly; keep an
 * allowlist for anything genuinely sensitive.
 */
const NEVER_LOG = new Set([
  "password", "passphrase", "token", "refreshToken", "accessToken",
  "authorization", "cookie", "secret", "jwt", "dek", "k_vault",
  "value", "label", "email", "phone", "pan", "aadhaar", "dob",
  "dateOfBirth", "address", "cgpa",
])

function scrub(obj: unknown, depth = 0): unknown {
  if (depth > 6) return "[deep]"
  if (Array.isArray(obj)) return obj.map((v) => scrub(v, depth + 1))
  if (obj && typeof obj === "object") {
    const out: Record<string, unknown> = {}
    for (const [k, v] of Object.entries(obj as Record<string, unknown>)) {
      out[k] = NEVER_LOG.has(k) ? "[redacted]" : scrub(v, depth + 1)
    }
    return out
  }
  if (typeof obj === "string" && obj.length > 512) return `${obj.slice(0, 512)}…`
  return obj
}

export const logger = pino({
  level: env.LOG_LEVEL,
  redact: { paths: [...NEVER_LOG].map((k) => `*.${k}`), censor: "[redacted]" },
  // The scrubber is the safety net for nested objects pino's paths miss.
  formatters: {
    log: (o) => scrub(o) as Record<string, unknown>,
  },
  transport: env.NODE_ENV === "development"
    ? { target: "pino-pretty", options: { colorize: true } }
    : undefined,
})

/** Never log these directly. Log the shape, never the content. */
export function logSync(userId: string, docCount: number, bytes: number) {
  logger.info({ userId, docCount, bytes }, "sync pull")
}
```

> **`logSync` logs the count and the byte size. Never the document keys.** A log line reading
> `{ docCount: 4, bytes: 812304 }` is operationally useful. A log line reading
> `{ keys: ["fullName", "pan", "aadhaar"] }` tells a stranger with log access what this user's
> documents contain. **Log the shape, never the content.**

```ts
// apps/api/src/server.ts
import { Hono } from "hono"
import { logger } from "./middleware/logger"
import { connectDb, disconnectDb } from "./db/connect"
import { env } from "./env"

const app = new Hono()
app.use("*", logger.middleware())

app.get("/health", (c) => c.json({ ok: true, ts: Date.now() }))

const server = Bun.serve // or: app.fire()

/**
 * Graceful shutdown. Without this, every deploy kills in-flight
 * requests mid-write and your sync logs fill with 502s.
 */
let shuttingDown = false
async function shutdown(signal: string) {
  if (shuttingDown) return
  shuttingDown = true
  logger.info({ signal }, "shutting down")
  await disconnectDb()
  process.exit(0)
}
process.on("SIGTERM", () => void shutdown("SIGTERM"))
process.on("SIGINT", () => void shutdown("SIGINT"))

await connectDb()
app.listen({ port: env.PORT })
logger.info({ port: env.PORT }, "api listening")
```

> **Graceful shutdown is not optional in a container platform.** Fly, Render, and Cloudflare
> all send `SIGTERM` and then wait a short grace period before `SIGKILL`. Without a handler,
> every single deploy truncates any request that was in flight. With four hundred users this is
> a few lost sync operations per week and a support inbox that says "sometimes it just doesn't
> save."

---

## Step 6 — Errors, not exceptions leaking

```ts
// apps/api/src/lib/errors.ts
export type ErrorCode =
  | "unauthorized" | "forbidden" | "not_found"
  | "rate_limited" | "validation_failed"
  | "conflict" | "internal"

export class AppError extends Error {
  constructor(
    readonly code: ErrorCode,
    message: string,
    readonly status: 400 | 401 | 403 | 404 | 409 | 422 | 429 | 500,
    readonly details?: unknown,
  ) {
    super(message)
    this.name = "AppError"
  }
}

// apps/api/src/middleware/errors.ts
export function onError(err: Error, c: Context) {
  if (err instanceof AppError) {
    return c.json({ error: { code: err.code, message: err.message, details: err.details } },
                  err.status as ContentfulStatusCode)
  }
  // Log the real error. Return something generic.
  logger.error({ err: err.message, stack: err.stack }, "unhandled")
  return c.json(
    { error: { code: "internal", message: "Something went wrong. Please try again." } },
    500,
  )
}
```

> **A stack trace in an API response is a free vulnerability report.** It reveals your file
> structure, your framework versions, your line numbers, and often an internal hostname. Log it;
> never return it.

---

## Step 7 — Rate limiting

```ts
// apps/api/src/middleware/rateLimit.ts
import { RateLimiterRedis } from "rate-limiter-flexible"

const authLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: "rl:auth",
  points: 5,            // 5 login attempts
  duration: 60,
})

const syncLimiter = new RateLimiterRedis({
  storeClient: redis,
  keyPrefix: "rl:sync",
  points: 120,          // sync is chatty by design
  duration: 60,
})

export async function limitAuth(c: Context, next: Next) {
  const key = c.req.header("x-forwarded-for")?.split(",")[0]?.trim() ?? "unknown"
  try {
    await authLimiter.consume(key)
  } catch {
    throw new AppError("rate_limited", "Too many attempts. Wait a minute.", 429)
  }
  await next()
}
```

> **Two limits, because they are two different problems.** `/auth/*` at 5/min is
> brute-force protection and must be tight. `/sync` at 120/min is because sync is chatty by
> design — a mobile client checking every 30 seconds uses 2/min, and a device waking from sleep
> with a 200-change backlog can burst. **Rate-limiting sync too tightly breaks the product on
> flaky mobile networks**, which is exactly the network your users are on.

---

## Step 8 — Verify before moving on

```bash
pnpm --filter @refrain/api dev
curl localhost:8787/health
# {"ok":true,"ts":1757012345678}
```

**Now test the things that will actually break:**

- [ ] `env.ts` throws on a missing `JWT_SECRET` at boot, not at request time
- [ ] Stopping and starting MongoDB gives a clear error, not a hang
- [ ] `SIGINT` shuts down cleanly and the process actually exits
- [ ] A thrown error returns `{ error: { code: "internal" } }` with **no stack**
- [ ] `logger.error({ email: "a@b.com" }, "x")` prints `email: "[redacted]"`
- [ ] `logger.info({ nested: { pan: "ABCDE1234F" } })` redacts the nested field
- [ ] 6 login attempts in a minute return 429
- [ ] A `.nic.in` request never reaches this server (no route, no log, no nothing)

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The §11 rewrite — the wording of every claim** | **You. 100%.** This is positioning and it is irreversible once published |
| **`NEVER_LOG` and the decision of what is sensitive** | **You.** You know what your product holds |
| `env.ts` and the Zod schema | **You** |
| `sanitizeFilter` and understanding why it matters | **You** |
| `maxPoolSize` reasoning | **You** |
| `connect.ts` boilerplate | **OpenCode** |
| `logger.ts` formatters | **OpenCode**, then verify the scrubber yourself against a nested case |
| Graceful shutdown handler | **OpenCode** |
| Rate limiter wiring | **OpenCode** — it needs Redis setup you have not done yet |
| `AppError` and the error middleware | **OpenCode** |

> **Redis is not installed yet.** For local dev, an in-memory limiter is fine and one fewer
> dependency is one fewer thing to run. Add Redis in Chapter 18 when you deploy, where
> multi-instance rate limiting actually requires it.

---

## Gotchas in this chapter

**`MongoServerSelectionError: timed out after 30000ms`.** Wrong URI, or no `mongoose.connect`
await. Set `serverSelectionTimeoutMS: 5000` so it fails in five seconds with a useful message
instead of thirty seconds of silence.

**The API works locally and 502s in production.** `CORS_ORIGINS` does not include your
production domain. Add it. This fails at the browser, in the console, with a CORS error and no
server log at all — because the request never arrives.

**The side panel cannot reach the API.** Your extension ID is not in `CORS_ORIGINS`. Add
`chrome-extension://<id>`.

**`MongooseError: Operation ... buffering timed out`.** You used a model before `connect()`
resolved. Await the connection in `app.ts` before mounting routes, or your first request 10s
later hits a model with no connection.

**User data appears in your logs.** Your scrubber only covers the top level. Log the count, not
the keys. Chapter 8's `logSync` shows the correct shape.

**One deploy killed a sync mid-write.** No `SIGTERM` handler. Chapter 18 sets the grace period
on the platform too.

**CI wiped your local database.** Shared database across branches. Use
`refrain_dev_$BRANCH`.

**`$ne: null` matched the wrong user.** `sanitizeFilter` is off. Turn it on in the connect
options, not per-query.

**Every request is 429 in development.** You copied the production limit and are refreshing the
page manually thirty times. Make the limit environment-dependent.

---

## Check yourself before Chapter 5

1. **What does a backend actually buy you here, in one sentence?**
2. **Which three §11 claims become false, and what is the rewrite for each?**
3. **Why is "we have no server" a worse line than "our server cannot read your data"?**
4. **What does a server that cannot read your data lose the ability to do?**
5. **Why must the API never import `@refrain/ui`?**
6. **What does `sanitizeFilter: true` prevent, concretely?**
7. **Why is the auth rate limit tight and the sync rate limit loose?**
8. **What must never appear in a log line, and why is the denylist still the wrong shape?**
9. **What happens on deploy without a `SIGTERM` handler?**

---

**Next: [Chapter 5 — The Data Model](./05-data-model.md)** — hybrid storage. Opaque blobs plus
a minimal non-sensitive metadata index, tombstones, and every Mongoose schema.