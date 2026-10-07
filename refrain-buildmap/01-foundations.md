# Chapter 1 — Foundations

> **Day 1 · Goal: a TypeScript project running on your machine that you can explain line by line.**
>
> No Refrain code yet. This chapter is about making the toolchain something you understand
> rather than something you tolerate.

---

## Understand this first

### What you are actually building

Three separate programs that ship together and share code:

| Program | Where it runs | Can it touch a form on google.com? |
|---|---|---|
| **Web app** | Normal browser tab, your domain | ❌ No. The same-origin policy forbids it. |
| **Side panel** | The thin strip beside your browser tabs | ✅ Yes — extensions can |
| **Content script** | Injected into every page you visit | ✅ Yes — but see the rule below |

The side panel and the web app are **the same React application** with two different entry
points. That is not laziness, it is the entire reason you can have a proper web UI at all.
Chrome 114+ turned the side panel into what is essentially a web page, so `packages/ui`,
your store, and your components are all shared. One codebase, two doors.

The content script is the third program and it is **the only one allowed to touch the page.**
It has no UI. No state. No business logic. It reads a form, and it writes values. Nothing else.

If you remember one thing from this chapter: **you cannot build this as a web app.** Not a
limitation you engineer around — a hard browser rule. A page on your domain has zero access
to the DOM of any other site.

### Why TypeScript and not JavaScript

Because your product's core value is **provenance** — knowing exactly where every value in
every field came from. That is a data-shape problem, and data-shape problems are what
TypeScript catches at compile time instead of at 2am in production.

Concretely, you will hit this in Chapter 2. A form field might be:

```ts
// This is a lie the type system should stop you telling:
{ label: "Name", value: 87.4 }   // ❌ value must be a string for an <input>

// This is what TypeScript forces you to confront:
{ label: "10th Percentage", value: "87.4", confidence: 0.98, source: "marksheet" }  // ✅
```

You will be mapping between two completely different worlds — typed Zod schemas in your
React UI, and the untyped, hostile, arbitrary DOM of a scholarship portal. Types are how you
survive that translation.

### What pnpm is, and why not npm

npm ships one copy of every dependency into a flat `node_modules` at the root of each project.
**pnpm uses a content-addressable store and symlinks.** If five packages all depend on
`zod@3`, you get one copy on disk and five symlinks.

For a monorepo this is the entire reason to use it. In Chapter 2 you will have five packages
importing from each other, and pnpm's strictness means **a package cannot import something it
did not declare in its own `package.json`.** That strictness is a feature: it is the reason
your dependency graph stays sane instead of becoming a haunted house.

### What a dev server actually is

Your code does not "run." A dev server watches your files, and when one changes it converts
JavaScript into something a browser can execute, serves it over HTTP, and pushes the update
into the open page without a refresh.

That last part — hot module replacement — is why you can work quickly. It is also why a
production build behaves differently from dev: dev is lenient and slow, production is strict
and fast. **Ship the production build, not the dev server.**

---

## Step 1 — Verify your toolchain

Do not skip this. Every "it doesn't work" bug in Chapter 7 is a version problem found here.

```bash
node -v      # need 20+ (Node 22 or 26 is ideal)
pnpm -v      # need 9+   (10.x is current)
git --version
```

If `pnpm` is missing:

```bash
corepack enable && corepack prepare pnpm@latest --activate
```

**Check your Chrome version — this gates the entire product:**

```bash
# Chrome stable, current as of 2026
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --version
```

| Chrome version | What you get |
|---|---|
| < 114 | ❌ No side panel. Product does not exist for you. |
| 114–137 | ✅ Side panel works. No built-in on-device LLM. |
| 138+ | ✅ Side panel **and** the `LanguageModel` API (Gemini Nano). Your primary inference engine is local. |

If you are below 138, that is not a blocker — Chapter 8 has an Ollama fallback that works
today. But know which engine you have before you start.

### Open Chrome with a clean profile

Extensions behave differently with your personal profile and your other extensions installed.
You want a clean, disposable profile for development.

1. Open a **Guest** window, or better, create a separate Chrome profile called `refrain-dev`
2. Never develop an extension in your daily-browsing profile

---

## Step 2 — Create your first project, by hand

We are doing this manually on purpose. `pnpm create vite` would scaffold it in one line, and
then you would be guessing what the eight files do.

```bash
mkdir -p ~/refrain-lab
cd ~/refrain-lab
git init
```

Now create **one file**, `hello.ts`:

```ts
const greet = (name: string): string => `Hello, ${name}`
console.log(greet("Refrain"))
```

That is real TypeScript. A typed parameter, an explicit return type, a template literal.

### Run it

```bash
npx tsx hello.ts
```

`tsx` runs TypeScript directly with no build step. You will use it constantly for quick
scripts. Install it globally once:

```bash
npm i -g tsx
tsx hello.ts
```

Output: `Hello, Refrain`

**Now break it on purpose.** Change the signature to accept a number, pass it a string, and
read the error. This is the actual lesson — TypeScript's errors are your friend and you need
to read them fast:

```ts
const greet = (name: number): string => `Hello, ${name}`
console.log(greet("Refrain"))
```

```
error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
```

That is the tool working. A JavaScript version of this program would not error — it would
silently produce `Hello, Refrain` anyway, and you would ship it.

### Understand what `package.json` actually is

Create it yourself so you never wonder again:

```json
{
  "name": "refrain-lab",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "tsx hello.ts",
    "typecheck": "tsc --noEmit"
  }
}
```

| Field | What it means |
|---|---|
| `private: true` | Stops you accidentally publishing. Every package in a monorepo needs this. |
| `type: "module"` | ES modules — the modern standard. Vite requires it. |
| `scripts` | Aliases. `pnpm start` runs `tsx hello.ts`. Never type raw commands twice. |

**That `scripts` block is the single highest-leverage habit in this project.** `pnpm typecheck`
instead of `tsc --noEmit` means the command never changes, and it never gets mistyped in
chapter 11 of your life at 1am.

### Understand `tsconfig.json`

```json
{
  "compilerOptions": {
    "target": "ES2022",
    "module": "ESNext",
    "moduleResolution": "bundler",
    "lib": ["ES2022", "DOM", "DOM.Iterable"],
    "strict": true,
    "noEmit": true,
    "noUncheckedIndexedAccess": true,
    "verbatimModuleSyntax": true,
    "skipLibCheck": true,
    "isolatedModules": true,
    "paths": { "@/*": ["./src/*"] }
  },
  "include": ["src"]
}
```

| Option | Why it is on |
|---|---|
| `strict` | **Non-negotiable.** The entire point. Every "why is TypeScript annoying" complaint is a `strict: false` codebase. |
| `noUncheckedIndexedAccess` | `arr[0]` returns `T \| undefined`. Your DOM queries and field arrays deserve that honesty. |
| `verbatimModuleSyntax` | Forces `import type` for types. Faster builds, no runtime surprises. |
| `moduleResolution: "bundler"` | How imports resolve in a Vite world. |
| `skipLibCheck` | Do not typecheck other people's code. Only yours. |

> **Version warning.** Older tutorials show `"jsx": "react-jsx"` in the root config. From
> Chapter 3 onward this file gets split into shared presets in `packages/tsconfig/`. Do not
> build this file out by hand again after Chapter 2 — you will replace it with a preset.

---

## Step 3 — Git, and your first safety net

```bash
git add .
git commit -m "chore: scratchpad to understand the toolchain"
```

Now create `.gitignore` **before** you write any real code:

```gitignore
node_modules/
dist/
.turbo/
build/
.env
.env.local
.DS_Store
*.log
```

**Why this file is the most important file in the project.** Your product reads marksheets,
signatures, and Aadhaar-linked addresses. You will `git init` and commit things you did not
mean to commit. This file is your first line of defence, and your §11 privacy claims are
only true if you are rigorous about it.

```bash
git commit -m "chore: gitignore before any real code exists"
```

**Read that commit history.** In six months, when you are hunting a regression, this log is
the difference between a five-minute bisect and an afternoon. Commit messages are future
notes to yourself.

---

## Step 4 — Clean up

```bash
rm -rf ~/refrain-lab
```

You now have the toolchain and you understand every file in it. Chapter 2 starts the actual
monorepo.

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| Install Node, pnpm, verify Chrome version | **You** |
| Write `tsconfig.json` from memory | **You** — this is the one to learn |
| Type the error-message experiments | **You** |
| Git init, first two commits | **You** |
| Confirm on-device LLM availability | **OpenCode** — paste your Chrome version, ask what the API surface is |

---

## Gotchas in this chapter

**`pnpm` installed but every command fails with `EACCES`.** On macOS, Node installed via a
`.pkg` from nodejs.org sets a different prefix than a Homebrew install. Fix:

```bash
echo 'export PATH="$(brew --prefix)/bin:$PATH"' >> ~/.zshrc && source ~/.zshrc
```

**`Cannot find module 'typescript'`.** `tsx` runs TypeScript but does not install it. You need
the compiler for `tsc --noEmit`:

```bash
pnpm add -D typescript
```

**Your editor is red everywhere after `strict: true`.** Expected — that is `strict` earning its
keep. Do not turn it off to make the red go away. Fix the code.

---

## Verify before moving on

You are done with Chapter 1 when **all** of these are true:

- [ ] `node -v` and `pnpm -v` report the versions above
- [ ] You know your Chrome version and whether you have the `LanguageModel` API
- [ ] You can explain what `type: "module"` does without looking it up
- [ ] You can explain every line of `tsconfig.json`
- [ ] You have read a TypeScript error and understood it without copying the fix
- [ ] You have a `.gitignore` in a repo with two commits in it

---

## Check yourself before Chapter 2

If you cannot answer these, reread the chapter. They will come up constantly.

1. **Why can't a web app fill a form on another website?** Name the rule.
2. **What is the only component allowed to touch the page, and what is it forbidden from doing?**
3. **You open a form and the extension reports zero fields. What is your first hypothesis?**
4. **What does `noUncheckedIndexedAccess` do, and why does your content script need it?**
5. **What breaks if you put an extension in your personal Chrome profile?**

---

**Next: [Chapter 2 — The Monorepo](./02-monorepo.md)** — pnpm workspaces, Turborepo,
`apps/` and `packages/`, and why `pnpm dev` will boot three programs at once.
