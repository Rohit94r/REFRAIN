# Chapter 18 — CI/CD & Deployment

> **Day 18 · Goal: one command ships all three surfaces, and you know how to undo it.**
>
> Three targets, three update cadences, and one shared dependency graph. Getting this wrong is
> how a user ends up on a two-week-old extension talking to a web app you deployed an hour ago.

---

## Words you need to know

I use these words in this chapter. I explain each one here in simple
words, so you do not have to guess.

- **CI** — continuous integration. Your code is checked automatically on every
  push.
- **CD** — continuous deployment. Passing code is shipped automatically.
- **GitHub Actions** — the tool that runs those checks.
- **Workflow** — one automated process, written as a YAML file.
- **YAML** — the format those config files use. Indentation matters.
- **Docker** — a way to package your app so it runs the same everywhere.
- **Container** — the packaged app, including everything it needs.
- **Deploy** — putting your code where users can reach it.
- **Rollback** — going back to the previous working version.
- **Migration** — changing a live database's shape without breaking it.
- **Secret** — a password or key. Stored in the platform, never in the code.
- **Extension review** — Chrome checks an extension before publishing. Takes 1 to
  14 days.

---

## Understand this first

### You have three deployables on three clocks

| Surface | Ships by | Reaches users | Rollback speed |
|---|---|---|---|
| **Extension** | Chrome Web Store review | **1–14 days** | Cannot be recalled at all |
| **Web app** | Any static host | Seconds | Seconds |
| **API** | Container platform | Minutes | Minutes |

**The extension is the one you cannot take back.** Once Chrome serves a version, users stay on
it until they update, and Chrome updates extensions silently in the background — some within
hours, some not for weeks. There is no "unpublish," no "hotfix," no recall.

This single fact drives most of this chapter. Every other deploy is reversible; that one is a
letter to the future.

### The consequence: the web app must be backward-compatible with every live extension

Your web app deploys in seconds. Your extension takes up to two weeks to reach users. So at
any moment, real users are running **every extension version you have ever shipped, forever**,
against the newest web app.

```
web v1.4  ← you deploy this now
   ▲
   ├── ext v1.0   ← 8% of users, will not update for 3 weeks
   ├── ext v1.1   ← 22% of users
   ├── ext v1.2   ← 41% of users
   └── ext v1.3   ← 29% of users, updated yesterday
```

**Your newest code must work with the oldest code you ever shipped.** Not the oldest you
currently support — the oldest that is still running in the wild, and it is running right now.

That is a hard constraint on how you evolve `packages/fields`, and it is not optional:

```ts
// ❌ Renaming a field breaks every extension in the wild.
//    An ext v1.2 sends { label: "..." }; your new web app
//    expects { question: "..." }. The review screen breaks for
//    22% of users and there is nothing you can do but ship a fix
//    extension and wait two weeks.
export const FormFieldSchema = z.object({ question: z.string() })

// ✅ Add, never rename or remove.
export const FormFieldSchema = z.object({
  label: z.string(),
  /** Added in v1.3. Optional so v1.0–v1.2 extensions still parse. */
  maxLength: z.number().int().positive().optional(),
})
```

The rules that follow, and they are strict:

| Change | Allowed? | Why |
|---|---|---|
| **Add** an optional field | ✅ | Old senders omit it; new senders' extra field is ignored by old receivers only if you strip unknown keys |
| **Add** a required field | ❌ | Old senders omit it → parse failure |
| **Rename** a field | ❌ | Every live extension breaks |
| **Remove** a field | ❌ | New senders break old receivers |
| **Narrow** a type or an enum | ❌ | Old senders now fail validation |
| **Widen** an enum | ✅ | Old receivers must tolerate unknown values |
| **Change** a `confidence` threshold | ⚠️ | Behaviour change, not a parse failure — ship it, but expect confusion |

> **Strip unknown keys on every message boundary.** `FormSchema.parse(raw)` in Zod **does** strip
> unknowns by default — which is exactly the behaviour you want, and exactly why you should not
> set `.strict()` on any schema that crosses the extension boundary. **`.strict()` is for your
> database, not your wire protocol.** Getting this backwards breaks every deployed extension
> within an hour.

**Practical consequence: schema changes need a version field and a deprecation window.** Add
`schemaVersion: 1` to every cross-boundary message, and only remove fields two minor versions
later. Chapter 8's `SYNC_VERSION` is the same idea applied to sync.

### Web Store review is a queue, not a step

Budget **1–14 days**, and treat 14 as real. Reviews reject for reasons that have nothing to do
with code:

- A permission you declared but do not obviously use
- A privacy disclosure that does not match your actual behaviour
- Marketing copy that promises something the listing does not show
- Missing or wrong privacy policy URL

**Ship early and often.** Every Web Store submission is a review. Your first submission should
be a minimal extension that does one thing, submitted in week one — not the finished product in
week fifteen. You want the review clock running while you build.

---

## Step 2 — GitHub Actions

### One workflow, three jobs, cache everything

```yaml
# .github/workflows/ci.yml
name: CI
on:
  push: { branches: [main] }
  pull_request:

concurrency:
  # A new push to a PR cancels the previous run. Nobody wants to
  # wait for stale commits to finish before their new one starts.
  group: ci-${{ github.ref }}
  cancel-in-progress: ${{ github.event_name == 'pull_request' }}

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: pnpm/action-setup@v4
        with: { version: 10 }

      - uses: actions/setup-node@v4
        with:
          node-version: 22
          cache: pnpm

      - run: pnpm install --frozen-lockfile

      - name: Check (typecheck → lint → test → build)
        run: pnpm check

      # The privacy greps, in CI, on every PR. If these can fail
      # locally they will eventually fail in a release.
      - name: Privacy gates
        run: bash scripts/privacy-gate.sh

  extension:
    needs: check
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build --filter=@refrain/extension

      - name: Verify the built manifest is clean
        run: |
          MANIFEST=apps/extension/.output/chrome-mv3/manifest.json
          grep -E 'https?://' "$MANIFEST" && { echo "✗ remote host in manifest"; exit 1; }
          node -e "
            const m = require('./$MANIFEST');
            if (m.manifest_version !== 3) throw new Error('not MV3');
            if (!m.host_permissions) throw new Error('no host_permissions declared');
            console.log('✓ manifest clean');
          "

      - uses: actions/upload-artifact@v4
        with:
          name: extension-chrome-mv3
          path: apps/extension/.output/chrome-mv3/
          retention-days: 30

  api:
    needs: check
    runs-on: ubuntu-latest
    services:
      mongodb:
        image: mongo:8
        ports: ['27017:27017']
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
      - with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build --filter=@refrain/api

      # Integration tests need a real MongoDB, not a mock. Mongoose
      # behaviour — indexes, transactions, query casting — is exactly
      # what mocks get wrong.
      - run: pnpm --filter @refrain/api test:integration
        env:
          MONGODB_URI: mongodb://localhost:27017
          MONGODB_DB: refrain_test
          JWT_SECRET: test-secret-that-is-long-enough-to-pass-validation-32
          JWT_PRIVATE_KEY_PEM: ${{ secrets.TEST_JWT_PRIVATE_KEY }}
          CORS_ORIGINS: http://localhost:5173
```

> **`pnpm install --frozen-lockfile` is not optional in CI.** Without it, CI silently resolves
> fresh versions when `package.json` and `pnpm-lock.yaml` disagree, which means your build
> passed with code that does not exist in your lockfile. Every "works on CI, broken locally"
> story I have traced started here.

### The privacy gate script

These are Chapter 17's greps, promoted from "remember to run" to "cannot merge without."

```bash
#!/usr/bin/env bash
# scripts/privacy-gate.sh — runs in CI on every PR. Must return clean.
set -uo pipefail
fail=0
gate() {
  if eval "$2" | grep -qE "$3"; then
    echo "✗ $1"; fail=1
  else
    echo "✓ $1"
  fi
}

# §11 no-list: nothing that submits forms or solves CAPTCHAs.
gate "no captcha solving" \
  "grep -rniE '2captcha|anticaptcha|capsolver|deathbycaptcha' packages/ apps/ --include='*.ts' --include='*.tsx'" \
  ""

gate "no automation drivers" \
  "grep -rniE 'selenium|puppeteer|playwright|webdriver' packages/ apps/ --include='*.ts' --include='*.tsx'" \
  "^$"

gate "no submit automation" \
  "grep -rniE '\\.click\\(\\)|requestSubmit\\(|\\.submit\\(\\)|form\\.submit' packages/ apps/ --include='*.ts' --include='*.tsx'" \
  "test|spec|__tests__"

gate "no telemetry" \
  "grep -rniE 'google-analytics|gtag|mixpanel|posthog|segment\\.com|amplitude|sentry' packages/ apps/ --include='*.ts' --include='*.tsx'" \
  "test|spec|__tests__"

# No remote hosts in the shipped extension.
gate "no remote hosts in the extension" \
  "cat apps/extension/.output/chrome-mv3/manifest.json 2>/dev/null" \
  "https?://"

# No remote URLs in the extension bundle.
gate "no remote URLs in the extension bundle" \
  "grep -rnoE 'https?://[a-zA-Z0-9./-]+' apps/extension/.output/chrome-mv3/ --include='*.js'" \
  "refrain\.dev|localhost|schemas|w3\.org"

# The sync API must never carry content in metadata.
gate "syncmeta declares no content fields" \
  "grep -nE 'label|title|filename|value|mimeType|email|pan|aadhaar' apps/api/src/db/models/syncmeta.ts" \
  "^$"

exit $fail
```

> **The `--include='*.ts'` on the no-driver grep matters.** Playwright is a *dev* dependency for
> your own end-to-end tests, and blocking it entirely would block the tests that verify your
> privacy gates. The gate checks **shipped source**, not test files, and says so in the pattern.
>
> **The `sentry` grep is scoped to `packages/` and `apps/` and excludes tests** — because
> Chapter 4 allows Sentry on the server and forbids it in the extension. A grep that cannot
> distinguish those two is a grep you will disable within a month. Add a second, narrower gate:
>
> ```bash
> gate "no monitoring in the extension" \
>   "grep -rniE 'sentry|bugsnag|datadog' apps/extension/ --include='*.ts'" \
>   "^$"
> ```

---

## Step 3 — Turborepo caching

`turbo.json` from Chapter 2 already declares the task graph. What changes in CI is *what gets
cached and where*.

```jsonc
// turbo.json
{
  "tasks": {
    "build": {
      "dependsOn": ["^build"],
      "inputs": [
        "$TURBO_DEFAULT$",
        ".env*",                 // ← only if you want env changes to bust the cache
        "!**/*.md",
        "!**/.env.example"
      ],
      "outputs": ["dist/**", ".output/**", ".wxt/**"]
    },
    "typecheck": { "dependsOn": ["^build"], "outputs": [] },
    "lint":      { "outputs": [] },
    "test":      {
      "dependsOn": ["^build"],
      "outputs": ["coverage/**"],
      "env": ["NODE_ENV"]
    },
    "dev": { "cache": false, "persistent": true }
  }
}
```

```bash
# Locally, see what CI would do and why it is slow or fast
pnpm turbo run build --dry=json > turbo-graph.json
pnpm turbo run build --summarize          # what was cached, what ran, why
```

**Where Turborepo caches in CI matters for your bill and your speed.**

```yaml
# GitHub-hosted runners: cache in the repo, keyed by lockfile + task.
- uses: actions/cache@v4
  with:
    path: .turbo/cache
    key: turbo-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-${{ hashFiles('turbo.json') }}
    restore-keys: turbo-${{ runner.os }}-${{ hashFiles('pnpm-lock.yaml') }}-
```

> **The cache key is the lockfile hash, and that is the entire design.** Change any dependency
> version → new key → full rebuild. Change a comment → same key → instant. Getting this wrong in
> either direction is bad: too coarse and every push is a 4-minute build; too fine and you
> rebuild everything on every commit, which is the same thing.
>
> **The restore-keys fallback matters on the first run after a dependency bump** — it pulls the
> previous cache so `node_modules`-adjacent work is warm even though the build itself re-runs.

---

## Step 4 — Environments and secrets

### Three environments, three databases, three key pairs

```
development  →  localhost:27017/refrain          →  dev keys (in .env, gitignored)
staging      →  Atlas refrain_staging            →  staging keys (GitHub environment secrets)
production   →  Atlas refrain                   →  prod keys (GitHub environment secrets)
```

```yaml
# GitHub environment protection — staging and prod require approval.
# A deploy is a human decision, not a push.
environments:
  staging:
  production:
```

> **Set required reviewers on the `production` environment.** Then a merge to `main` builds and
> tests everything, but the actual production deploy waits for a human to approve it. This is one
> setting that prevents the class of outage where a green PR contains something nobody
> reviewed, and it costs nothing.
>
> **Staging must have its own JWT keypair and its own database.** Sharing either with production
> means staging does not test production, and a leaked staging token is a production credential.

### The secrets list

| Secret | Where | Rotate by |
|---|---|---|
| `JWT_PRIVATE_KEY_PEM` | GitHub env secret | Every 6 months, with `kid` overlap |
| `JWT_PUBLIC_KEY_PEM` | GitHub env secret (non-secret but keep together) | With the private key |
| `MONGODB_URI` | GitHub env secret | On credential rotation |
| `MONGODB_DB` | GitHub env variable (not secret) | — |
| `CORS_ORIGINS` | GitHub env variable | When adding a domain |
| `FLY_API_TOKEN` | GitHub env secret | Per deploy |
| `CHROME_EXTENSION_ID` | GitHub env variable | Once published |

```bash
# The extension public key is NOT a secret — it is published at
# /.well-known/jwks.json. Shipping it in the bundle is the point.
# This trips secret scanners every time and needs an allowlist entry.
```

> **`JWT_PUBLIC_KEY` in a Chrome extension bundle will trip every secret scanner.** It is not a
> secret; it is a *public* key that must be verifiable offline (Chapter 6). Add a scanner
> allowlist entry with a comment explaining why, or you will "fix" it by removing the key and
> break offline verification during a scanner-driven cleanup.

### Never in git, ever

```gitignore
.env
.env.*
!.env.example
*.pem
apps/extension/.env
apps/extension/.output/
apps/extension/.wxt/
.turbo/
node_modules/
```

```bash
# Prove it, in CI, on every push:
git ls-files | grep -E '\.env$|\.pem$' && { echo "✗ secret committed"; exit 1; } || echo "✓ no secrets in git"
```

---

## Step 5 — MongoDB has no schema migrations, and that is not the same as "no migrations"

This trips everyone once. SQL has migrations because the schema is coupled to the data. MongoDB
decouples them, so you have **three different kinds of change** and each needs its own process.

### Kind 1 — index changes

Indexes are not created by `insert()`. They must exist before a query relies on them, and a
missing index on a growing collection is a silent production incident.

```ts
// apps/api/src/db/migrations/indexes.ts
/**
 * Idempotent index creation. Safe to run on every boot.
 *
 * Mongoose's autoIndex does this too, but it is DISABLED in
 * production by default (correctly — an accidental index build on
 * boot can lock a collection for minutes). So you do it explicitly.
 */
export async function ensureIndexes() {
  await Promise.all([
    UserModel.syncIndexes(),
    BlobModel.syncIndexes(),
    SyncMetaModel.syncIndexes(),
    DeviceModel.syncIndexes(),
    RefreshTokenModel.syncIndexes(),
    CountersModel.syncIndexes(),
    OpModel.syncIndexes(),
  ])
}

// Called once, after connectDb(), before app.listen()
await connectDb()
await ensureIndexes()
app.listen({ port: env.PORT })
```

> **`syncIndexes()` drops indexes that are not in the schema.** That is what you want — a renamed
> index would otherwise linger forever consuming write performance — but it means **removing a
> field from a schema silently drops its index.** Before you rely on a dropped index, check
> `explain()`. This has bitten people who removed a `legacy` field from a model and did not
> notice a query went from indexed to collection-scan overnight.

### Kind 2 — data backfills

Adding a field to existing rows:

```ts
// apps/api/src/db/migrations/backfill-v2.ts
/**
 * Example: facts uploaded before v2 have no `kdfVersion`. Every
 * row needs one before the new code can read them.
 *
 * Run ONCE, in a script, not on boot. Backfills on boot are how
 * you take down production with a slow query at the worst moment.
 */
export async function backfillKdfVersion() {
  const result = await BlobModel.updateMany(
    { kdfVersion: { $exists: false } },
    { $set: { kdfVersion: 1 } },
  )
  console.log(`backfilled ${result.modifiedCount} blobs`)
}
```

```bash
pnpm --filter @refrain/api migrate:backfill-kdf
```

> **Batch every update.** `updateMany` on 500,000 documents in one command holds locks and can
> exceed the oplog window, which on a replica set **rolls back the primary**. Chunk it:
>
> ```ts
> for await (const doc of BlobModel.find({ kdfVersion: { $exists: false } }).limit(1000)) {
>   await BlobModel.updateOne({ _id: doc._id }, { $set: { kdfVersion: 1 } })
> }
> ```
>
> A backfill that takes 40 seconds is fine. One that takes 12 minutes while you watch replicas
> fall out of the sync set is a different afternoon.

### Kind 3 — protocol migrations (the one that really matters)

The sync protocol version is the migration with real user impact.

```ts
// apps/api/src/db/migrations/sync-v2.ts
/**
 * Bump SYNC_VERSION from 1 to 2 because the fact document shape
 * changed: v1 had { key, value, confidence } and v2 adds
 * { schemaVersion: 2, ... }.
 *
 * The server does NOT rewrite existing blobs. It cannot — they are
 * ciphertext. Instead it bumps every existing syncmeta row's
 * revision so every client sees its documents as changed, pulls
 * them, finds the shape wrong, and re-uploads in the new shape.
 */
export async function prepareSyncV2() {
  // Mark every user's docs as changed. Cheap: one updateMany.
  const users = await SyncMetaModel.distinct("userId")
  for (const userId of users) {
    const counter = await CountersModel.findOneAndUpdate(
      { _id: userId }, { $inc: { value: 1000 } }, { new: true, upsert: true },
    )
    await SyncMetaModel.updateMany(
      { userId, deleted: false },
      { $set: { revision: counter.value } },
    )
  }
  // Bump last, so a crash mid-way leaves the old version live
  // rather than clients seeing a version the data does not match.
  process.env.SYNC_VERSION = "2"
}
```

> **The `+1000` revision jump is deliberate.** If you bump revisions one at a time, a client
> syncing during the migration pulls some docs at the new revision and some at the old, and
> ends up with a **half-migrated vault** — some facts readable, some not. Jumping every document
> past the current cursor in one move means clients see *either* everything changed or nothing.
> One migration, one boundary, no half-states.

---

## Step 6 — Deploying

### API

```dockerfile
# apps/api/Dockerfile
FROM node:22-alpine AS base
RUN corepack enable && corepack prepare pnpm@10 --activate
WORKDIR /app

# ── deps: copied separately so a source change does not reinstall ──
FROM base AS deps
COPY pnpm-lock.yaml pnpm-workspace.yaml package.json ./
COPY apps/api/package.json apps/api/
RUN pnpm install --frozen-lockfile --filter @refrain/api...

# ── build ──
FROM base AS build
COPY --from=deps /app/node_modules ./node_modules
COPY . .
RUN pnpm turbo run build --filter=@refrain/api

# ── run ──
FROM base AS run
ENV NODE_ENV=production
RUN addgroup -S refrain && adduser -S refrain -G refrain
COPY --from=build --chown=refrain:refrain /app/apps/api/dist ./dist
COPY --from=build --chown=refrain:refrain /app/node_modules ./node_modules
USER refrain
EXPOSE 8787
HEALTHCHECK --interval=30s --timeout=3s --start-period=10s \
  CMD wget -qO- http://localhost:8787/health/ready || exit 1
CMD ["node", "dist/server.js"]
```

> **Run as a non-root user.** `USER refrain` is one line and it is the difference between a
> container escape being catastrophic and being annoying. It is the first thing a security
> review will check and the easiest thing to get wrong when you are copying a Dockerfile.
>
> **`--chown=refrain:refrain` on the copy, not a later `chown -R`.** Recursive chowns on large
> `node_modules` directories add 10+ seconds to every build for no benefit.

```bash
# fly.toml
app = "refrain-api"
primary_region = "bom"          # Mumbai. Your users are in India.

[build]
  dockerfile = "apps/api/Dockerfile"

[http_service]
  internal_port = 8787
  force_https = true
  auto_stop_machines = "stop"
  auto_start_machines = true
  min_machines_running = 1

[[http_service.checks]]
  interval = "15s"
  timeout = "3s"
  grace_period = "10s"
  method = "GET"
  path = "/health/ready"

[env]
  NODE_ENV = "production"
  PORT = "8787"
  MONGODB_DB = "refrain"
  CORS_ORIGINS = "https://refrain.dev,https://api.refrain.dev"
```

> **Region choice is a product decision, not an ops one.** Mumbai (`bom`) versus Virginia (`iad`)
> is roughly 200ms of round trip. For a sync API that fires on `visibilitychange` and `online`,
> 200ms is the difference between sync feeling instant and feeling broken. **Put the server where
> your users are.** And for a country with unreliable mobile networks, 200ms matters more than it
> does for users in a datacenter city.

### Web app

```yaml
# .github/workflows/deploy-web.yml
name: Deploy web
on: { workflow_run: { workflows: [CI], branches: [main], types: [completed] } }
jobs:
  deploy:
    if: ${{ github.event.workflow_run.conclusion == 'success' }}
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
        with: { ref: ${{ github.event.workflow_run.head_sha }} }
      - uses: pnpm/action-setup@v4
        with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build --filter=@refrain/web

      # History fallback. createBrowserRouter needs every path to
      # serve index.html or a hard refresh on /profile 404s.
      - run: bash scripts/deploy-web.sh
```

```bash
# scripts/deploy-web.sh
npx wrangler pages deploy apps/web/dist \
  --project-name refrain-web \
  --branch main
```

```jsonc
// public/_redirects — the history fallback, on every static host
/*  /index.html  200
```

> **The `_redirects` rule is the single most common static-host bug**, and it only shows up on a
> hard refresh of a nested route. A user bookmarks `/profile`, opens it next week, gets a 404,
> and concludes your app is broken. Chapter 11's gotcha list already flagged this; this is where
> it gets fixed.

### Extension

```bash
# Build. Deterministic, and it is a zip because that is what the
# Web Store accepts — not a deploy.
pnpm turbo run build --filter=@refrain/extension
cd apps/extension/.output/chrome-mv3 && zip -r ../../refrain-1.4.0.zip . -x '.*'
```

```yaml
# .github/workflows/release-extension.yml
name: Release extension
on:
  push:
    tags: ['ext-v*']
jobs:
  publish:
    environment: production
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: pnpm/action-setup@v4
        with: { version: 10 }
      - uses: actions/setup-node@v4
        with: { node-version: 22, cache: pnpm }
      - run: pnpm install --frozen-lockfile
      - run: pnpm turbo run build --filter=@refrain/extension
      - run: bash scripts/privacy-gate.sh

      - name: Version the manifest from the tag
        run: |
          M=apps/extension/.output/chrome-mv3/manifest.json
          jq --arg v "${GITHUB_REF_NAME#ext-v}" '.version = $v' "$M" > "$M.tmp" && mv "$M.tmp" "$M"

      - name: Upload to the Web Store
        run: |
          npx chrome-webstore-upload-cli \
            --source apps/extension/.output/chrome-mv3 \
            --extension-id "$CHROME_EXTENSION_ID" \
            --client-id "$GOOGLE_CLIENT_ID" \
            --client-secret "$GOOGLE_CLIENT_SECRET" \
            --refresh-token "$GOOGLE_REFRESH_TOKEN" \
            --auto-publish

      - uses: actions/upload-artifact@v4
        with: { name: extension-zip, path: refrain-*.zip }
```

**Tags, not `main` pushes, trigger an extension release.** Tags mean you decide when to ship —
you are not at the mercy of whatever landed on `main` that afternoon.

> **Version the manifest from the tag, never from `package.json`.** They drift the moment someone
> bumps one and not the other, and Chrome displays whatever is in the manifest. Bumping from the
> tag makes the tag the single source of truth, and `ext-v1.4.0` in git is the version in the
> store — greppable, auditable, one number.
>
> **Extension versions must strictly increase to be accepted.** Chrome rejects a re-upload with
> the same version number, and there is no "unpublish the bad one" — a rejected version is simply
> gone.

---

## Step 7 — Rollback

| What broke | Rollback | Time | Data risk |
|---|---|---|---|
| **Web app** deploy | Redeploy previous SHA | ~60s | None |
| **API** deploy | `fly deploy --image <previous>` | ~90s | **Possible** — see below |
| **Extension** deploy | **Cannot.** Must ship v1.4.1 | 1–14 days | Depends on severity |
| **Bad migration** | Restore from PITR | Minutes | **Yes** — see below |

```bash
# Web: instant. The previous SHA is still in git.
npx wrangler pages deploy apps/web/dist --branch main --commit-hash "$PREVIOUS_SHA"

# API: previous image, same config.
fly deploy --image registry.fly.io/refrain-api:$(curl -s https://api.fly.io/v1/apps/refrain-api/releases | jq -r '.[] | select(.status=="succeeded") | .version' | head -2 | tail -1)
```

> **A code rollback is not a data rollback.** If the bad deploy ran a migration, redeploying the
> old code over new data can make things worse — the old code does not know about the new field
> and may reject the rows. **The rule: roll back code first to stop the bleeding, then decide
> about data separately.** Do not attempt a combined "roll back everything" during an incident.

### The 1am runbook

```
1. Is it the web app, the API, or the extension?       → check /health/ready, then the store
2. Roll back the newest deploy. Don't debug first.      → 90 seconds, usually ends it
3. Is data damaged?                                      → Atlas PITR, if yes
4. If it's the extension: ship 1.4.1 today.              → say so publicly within the hour
5. Write down what happened while it is fresh.          → 20 words beats 400 next week
```

> **Step 4 is the one people skip and it is the one that matters for this product specifically.**
> A broken extension means students cannot fill scholarship forms, and they will find out. A
> public status line saying *"version 1.4.1 is in review, expected within X"* costs you nothing
> and is the difference between "this project is abandoned" and "this project had a bug".
>
> **For a solo project, the runbook is the whole incident-response plan.** There is no on-call
> rotation, no status page infrastructure, no escalation policy. You are the entire response.
> Writing these five steps down *before* you need them is what makes step 2 happen in 90 seconds
> instead of 90 minutes.

```jsonc
// Atlas point-in-time — your only real data safety net.
{
  "name": "refrain",
  "backupEnabled": true,
  "pitrEnabled": true,
  "pitrRetentionHours": 168,       // 7 days
  "majorVersion": 8
}
```

> **PITR is free on every Atlas tier and it is the difference between "we lost data" and "we
> lost an afternoon."** Enable it before you have any users, because the first time you will
> want it is the first time something breaks, and configuring it under pressure is how you
> configure it wrong.
>
> Note what it can and cannot restore: **it restores the database, which is opaque ciphertext.**
> If a bug corrupted a *document* rather than the database, PITR cannot help — the ciphertext
> was correctly written with a wrong plaintext inside. **Client-side versioning is your only
> defence there**, which is another argument for Chapter 8's revision model keeping history.

---

## Step 8 — Monitoring

```ts
// apps/api/src/middleware/metrics.ts
import { Counter, Histogram } from "prom-client"

export const requests = new Counter({
  name: "refrain_requests_total",
  help: "Requests by route and status",
  labelNames: ["method", "route", "status"],
})

export const latency = new Histogram({
  name: "refrain_request_duration_seconds",
  help: "Request latency",
  labelNames: ["method", "route"],
  buckets: [0.01, 0.05, 0.1, 0.5, 1, 2, 5],
})

export const syncOps = new Counter({
  name: "refrain_sync_ops_total",
  help: "Sync operations by outcome",
  labelNames: ["op", "status"],   // op: push|pull|blobs, status: applied|duplicate|conflict
})
```

```ts
// Only three dashboards matter at this scale.
const ALERTS = [
  { name: "API down",        query: "up == 0",                       for: "2m" },
  { name: "Errors spiking",  query: "rate(errors_5xx[5m]) > 0.05",   for: "5m" },
  { name: "Sync stalling",   query: "rate(sync_conflict[1h]) > 0.2", for: "15m" },
]
```

**And the alert that is specific to this product:**

```ts
// If this fires, something is reading blobs it should not be.
/** A new route touching the ciphertext column. Add to ALERTS. */
{ name: "Blobs read outside /sync/blobs",
  query: "rate(db_blobs_reads[5m]) - rate(sync_blobs_reads[5m]) > 0" }
```

> **Monitor the shape, never the content — including in metrics.** Metric labels are strings in
> a time-series database with a different access-control story than your app. `docId: "email"` as
> a label means anyone with dashboard access knows your fact keys. Use the doc *type*, never the
> doc id. This is the log-scrubbing rule from Chapter 4 applied to a system you did not build.
>
> **The conflict-rate alert is the one that predicts a support ticket.** A rising
> `sync_conflict` rate means two devices are fighting over the same fact, and before long
> someone writes in saying "Refrain keeps asking me to pick between two versions of my CGPA."
> You would rather see the graph move three days before that email.

---

## Step 9 — What ships, in what order

Deploy order matters and it is not the order the products were built.

```
1. API schema additions (backwards compatible)     →  nothing breaks
2. API code reading the new field defensively      →  old clients still work
3. Web app                                          →  seconds
4. Extension, to the store                         →  days to weeks
5. API code depending on the new client behaviour   →  only after most clients updated
```

> **Never deploy step 5 before step 4 has actually landed.** "Most clients updated" means
> instrumented numbers, not optimism. Gate any server behaviour change on a real adoption
> threshold:
>
> ```ts
> // Only require the new field from clients that we know send it.
> const REQUIRES_SCHEMA_V2_MIN_VERSION = "1.3.0"
> if (clientVersionGte(extVersion, REQUIRES_SCHEMA_V2_MIN_VERSION)) {
>   requireField(body, "maxLength")
> }
> ```
>
> The alternative — flipping server behaviour on deploy day — breaks every user who has not
> updated their extension yet, which on day one is **all of them**.

---

## Step 10 — Commit

```bash
git add -A
git commit -m "ci: matrix workflow, privacy gates, docker, fly.toml, extension release on tag"
```

Add two rows to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | Schema changes are additive-only for one minor version | A shipped extension cannot be recalled. Users will run every version ever released, against the newest web app, forever. |
| 2026-10-XX | Extension releases are tag-triggered, not `main`-triggered | Shipping is a human decision. A version number that drifts from the tag is a version nobody can find. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The three-clocks table and what it forces** | **You.** This is the architecture |
| **The additive-only schema rule** | **You.** Getting this wrong breaks users for two weeks |
| **The region choice (Mumbai)** | **You** |
| **The 1am runbook** | **You** |
| **The `+1000` revision jump** | **You** |
| **`ensureIndexes` and the `syncIndexes` footgun** | **You** |
| Deploy ordering | **You** |
| `scripts/privacy-gate.sh` | **You** — then let OpenCode tighten the patterns |
| The CI workflow YAML | **OpenCode** |
| Dockerfile and `fly.toml` | **OpenCode** |
| Metrics definitions | **OpenCode** |
| Rollback commands | **OpenCode** — then **actually run one in staging** |

> **Run a rollback in staging before you need it.** A rollback script you have never executed
> is a hypothesis. Execute it, confirm it works, and note how long it actually took. It will be
> 90 seconds, not the 20 minutes you assume, and you will have the number in advance.

---

## Gotchas in this chapter

**A user hard-refreshes `/profile` and gets a 404.** Missing `_redirects` / history fallback.

**CI passes with code that does not exist in your lockfile.** Missing `--frozen-lockfile`.

**The Docker build takes 6 minutes.** `COPY . .` before `pnpm install`. Copy the manifests, install, then copy source. Order matters for the layer cache.

**`node dist/server.js` runs as root.** Missing `USER refrain`.

**The API works locally and 502s in production.** `CORS_ORIGINS` missing the production domain. It fails at the browser with no server log, because the request never arrives.

**The extension submits and Chrome rejects it: "permission not justified."** You declared `host_permissions: <all_urls>` and the listing does not explain why. Chapter 20's Web Store section.

**A deployment killed a sync mid-write.** No `SIGTERM` grace period on the platform.

**The Web Store review takes 13 days.** It does. Start the clock in week one.

**You renamed a field in `packages/fields` and 22% of users broke.** Additive-only. See above.

**The API dropped to 0 replicas during a 4-second Atlas blip.** Your liveness check queries MongoDB, so a database blip restarts every instance. Liveness must not check the database.

**`turbo` cached a build with a stale dependency.** `inputs` is missing `$TURBO_DEFAULT$`.

**A secret scanner flagged `JWT_PUBLIC_KEY` in the extension bundle and someone removed it.** Offline token verification broke. Add the allowlist entry with a comment.

**A backfill rolled back the primary.** Unbatched `updateMany` on a large collection. Chunk it.

**CI passed on a PR but the branch was 40 commits behind main.** Add `branches: [main]` to the deploy trigger and require a green main before release.

---

## Verify before moving on

- [ ] CI runs typecheck, lint, test, and build on every PR
- [ ] `pnpm install --frozen-lockfile` in every CI job
- [ ] Privacy gate script fails the build on each of its 8 checks (test each one)
- [ ] `git ls-files | grep -E '\.env|\.pem'` returns nothing
- [ ] The built extension manifest has no remote hosts
- [ ] Turborepo cache is keyed on the lockfile
- [ ] Production environment requires manual approval
- [ ] Staging and production have separate databases and separate JWT keys
- [ ] `/health` and `/health/ready` behave differently on a DB outage
- [ ] Docker image runs as a non-root user
- [ ] Liveness does **not** check MongoDB; readiness does
- [ ] A web rollback completes in under 2 minutes in staging
- [ ] Atlas PITR is enabled
- [ ] `_redirects` makes a hard refresh of `/profile` work
- [ ] Extension version comes from the git tag, and matches the store
- [ ] You can state which three schema changes are still legal

---

## Check yourself before Chapter 19

1. **Why can you roll back the web app but not the extension?**
2. **What rule does "you cannot recall a shipped extension" impose on `packages/fields`?**
3. **Why must liveness not check MongoDB?**
4. **Why is Mumbai the right region for this product?**
5. **Why does a sync-protocol migration jump revisions by 1000?**
6. **Why is `pnpm install --frozen-lockfile` required in CI?**
7. **Why is the Dockerfile's `COPY` order before `pnpm install` important?**
8. **What does a code rollback *not* undo?**
9. **Why does adding a required field to a cross-boundary schema break users you cannot reach?**
10. **Why should extension releases be tag-triggered?**

---

**Next: [Chapter 19 — Production Hardening](./19-production-hardening.md)** — the review
surface: performance, accessibility, the security headers that matter, load testing, backup and
disaster recovery, and the honest limits of what a solo project can operate.