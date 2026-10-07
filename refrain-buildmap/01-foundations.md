# Chapter 1 — Foundations

> **Day 1 · Goal: a TypeScript project running on your machine that you can explain line by line.**
>
> No Refrain code yet. This chapter is about making the toolchain something you understand
> rather than something you tolerate.

---

## Understand this first

### What you are actually building

Three separate programs that ship together and share code:

| Program            | Where it runs                           | Can it touch a form on google.com?       |
| ------------------ | --------------------------------------- | ---------------------------------------- |
| **Web app**        | Normal browser tab, your domain         | ❌ No. The same-origin policy forbids it. |
| **Side panel**     | The thin strip beside your browser tabs | ✅ Yes — extensions can                   |
| **Content script** | Injected into every page you visit      | ✅ Yes — but see the rule below           |

The side panel and the web app are **the same React application** with two different entry
points. That is not laziness, it is the entire reason you can have a proper web UI at all.
Chrome 114+ turned the side panel into what is essentially a web page, so `packages/ui`,
your store, and your components are all shared. One codebase, two doors.

The content script is the third program and it is **the only one allowed to touch the page.**
It has no UI. No state. No business logic. It reads a form, and it writes values. Nothing else.

If you remember one thing from this chapter: **you cannot build this as a web app.** Not a
limitation you engineer around — a hard browser rule. A page on your domain has zero access
to the DOM of any other site.

> **Naming note, because it will confuse you in Chapter 2.** "Three programs" and "two apps"
> are both true, and they are not in conflict. You get **two build outputs** — `apps/web` and
> `apps/extension` — because Chrome's WXT toolchain models entry points as *files inside one
> app*, not as separate apps. So the side panel and the content script are two entry points
> inside `apps/extension`. Three things ship; two packages produce them. Chapter 2 draws this.

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

**What you should see** (this is a real output from the machine this map was written on):

```
node: v26.10.0
pnpm: 10.32.1
git: git version 2.54.0 (Apple Git-157)
```

If `pnpm` is missing:

```bash
corepack enable && corepack prepare pnpm@latest --activate
```

> **Use `corepack`, not `npm i -g pnpm`.** Corepack ties the pnpm version to your
> `packageManager` field in `package.json`, so your machine, your CI, and your teammate all
> get the *same* pnpm. Installing pnpm globally is how you end up with a lockfile that one
> machine writes and another cannot read.

**Check your Chrome version — this gates the entire product:**

```bash
# Chrome stable, current as of 2026
"/Applications/Google Chrome.app/Contents/MacOS/Google Chrome" --version
```

| Chrome version | What you get                                                                                        |
| -------------- | --------------------------------------------------------------------------------------------------- |
| < 114          | ❌ No side panel. Product does not exist for you.                                                    |
| 114–137        | ✅ Side panel works. No built-in on-device LLM.                                                      |
| 138+           | ✅ Side panel **and** the `LanguageModel` API (Gemini Nano). Your primary inference engine is local. |

**What you should see** on a current machine:

```
Google Chrome 154.0.8037.98
```

**You are on 138+, so you have the `LanguageModel` API.** That is the good path: Chapter 8's
inference engine runs locally in Chrome, and you never touch a cloud model.

If you are below 138, that is not a blocker — Chapter 8 has an Ollama fallback that works
today. But know which engine you have before you start, because it changes what you test.

> **`--version` on a browser binary is safe.** It prints the version and exits without opening
> a window or touching your profile. Some Chrome builds will launch briefly; that is harmless.

### Confirm the `LanguageModel` API is actually there

Do not assume it from the version number. Check:

```bash
# Open Chrome, go to chrome://flags, enable "Optimization Guide On Device Model"
# if it is off, then check the real answer in DevTools console on any page:
```

```js
// Paste into DevTools console on any https page
typeof LanguageModel !== "undefined"
// → true means Chapter 8 can use the built-in model
```

**If that is `false`, you need the Ollama fallback.** Note it and move on — do not spend
Chapter 1 debugging Chrome flags.

### Open Chrome with a clean profile

Extensions behave differently with your personal profile and your other extensions installed.
You want a clean, disposable profile for development.

1. Open a **Guest** window, or better, create a separate Chrome profile called `refrain-dev`
2. Never develop an extension in your daily-browsing profile

> **Why this is not superstition.** A profile with 30 extensions injects 30 content scripts
> into every page. When your scanner reports 14 fields on Day 8, you will spend two hours
> debugging another extension's DOM mutations. Chapter 18's "does the page feel slower"
> measurement is also impossible on a personal profile — the delta is noise.
>
> **Guest mode is not enough**, because it disables your extensions, including the one you are
> developing. You need a *named profile* with Refrain loaded and everything else off.

---

## Step 2 — Create your first project, by hand

We are doing this manually on purpose. `pnpm create vite` would scaffold it in one line, and
then you would be guessing what the eight files do.

```bash
mkdir -p ~/refrain-lab/src
cd ~/refrain-lab
git init
```

> **Note `src/` in the `mkdir`.** That looks like a typo and it is the one thing in this step
> people get wrong. Your `tsconfig.json` sets `"include": ["src"]`, and TypeScript uses that
> to decide which files to check. If you create `hello.ts` at the repository root instead, the
> typechecker will not see it — and it will tell you so with a confusing message. See Step 4.

Now create **one file**, `src/hello.ts`:

```ts
const greet = (name: string): string => `Hello, ${name}`
console.log(greet("Refrain"))
```

That is real TypeScript. A typed parameter, an explicit return type, a template literal.

### Run it

```bash
npx tsx src/hello.ts
```

`tsx` runs TypeScript directly with no build step. You will use it constantly for quick
scripts. Install it locally rather than globally:

```bash
pnpm add -D tsx
pnpm exec tsx src/hello.ts
```

**What you should see:**

```
Hello, Refrain
```

> **Why local and not `npm i -g tsx`.** A global `tsx` is a version you do not control, on a
> machine you will forget about. Chapter 17's CI installs with `--frozen-lockfile`, which
> means a globally-installed tool is invisible to CI and invisible to `pnpm exec`. **Anything
> in your `scripts` block must be a local dependency**, or it works for you and nowhere else.

### Now break it on purpose

Change the signature to accept a number, pass it a string, and read the error. **This is the
actual lesson** — TypeScript's errors are your friend and you need to read them fast:

```ts
const greet = (name: number): string => `Hello, ${name}`
console.log(greet("Refrain"))
```

```bash
pnpm exec tsc --noEmit
```

**What you should see — this exact output is verified:**

```
src/hello.ts(3,19): error TS2345: Argument of type 'string' is not assignable to parameter of type 'number'.
```

Read it as a sentence: *at line 3, column 19, the thing you passed is a `string`, but the
function wants a `number`.* Column 19 is the `"Refrain"` argument.

> **A JavaScript version of this program would not error** — it would happily print
> `Hello, Refrain` anyway, and you would ship it. In production that class of bug is
> `undefined` in a form field, which is the single most common cause of the Chapter 7
> "filled but nothing arrived" bug. **This is not academic.** It is the exact failure your
> product must not have.

### Learn one more error before moving on

```ts
const greet = (name: number): string => `Hello, ${name}`
const labels = ["Name", "Email"]
const first: number = labels[0]
console.log(greet("Refrain"), first)
```

```bash
pnpm exec tsc --noEmit
```

```
src/hello.ts(3,14): error TS2322: Type 'string | undefined' is not assignable to type 'number'.
  Type 'undefined' is not assignable to type 'number'.
```

**That `| undefined` is `noUncheckedIndexedAccess` doing its job.** TypeScript is telling you
that `labels[0]` is *probably* a string, but on an empty array it is `undefined`, and you have
not proven that isn't the case.

Now confirm the flag is what did it:

```bash
pnpm exec tsc --noEmit --noUncheckedIndexedAccess false
```

```
src/hello.ts(3,14): error TS2322: Type 'string' is not assignable to type 'number'.
```

**Different error, and that difference is the whole flag.** With it off, TypeScript trusts you.
With it on, it makes you check. Your content script walks DOM queries and field arrays where an
off-by-one is normal, and this is the flag that turns a silent `undefined` into a compile error.

> **Delete both bad lines before you continue.** Leave `hello.ts` as the four-line correct
> version from the top of this step.

### Understand what `package.json` actually is

Create it yourself so you never wonder again:

```json
{
  "name": "refrain-lab",
  "version": "0.0.0",
  "private": true,
  "type": "module",
  "scripts": {
    "start": "tsx src/hello.ts",
    "typecheck": "tsc --noEmit"
  },
  "devDependencies": {
    "tsx": "^4.20.0",
    "typescript": "^5.9.0"
  }
}
```

| Field            | What it means                                                              |
| ---------------- | -------------------------------------------------------------------------- |
| `private: true`  | Stops you accidentally publishing. Every package in a monorepo needs this. |
| `type: "module"` | ES modules — the modern standard. Vite requires it.                        |
| `scripts`        | Aliases. `pnpm start` runs `tsx src/hello.ts`. Never type raw commands twice. |
| `devDependencies` | Build-time only. Nothing here ships to a user.                             |

**That `scripts` block is the single highest-leverage habit in this project.**
`pnpm typecheck` instead of `tsc --noEmit` means the command never changes, and it never gets
mistyped in chapter 11 of your life at 1am.

Verify it works:

```bash
pnpm start
pnpm typecheck
```

`pnpm typecheck` should print nothing at all. **Silence is success.** If you see output from a
typecheck that is supposed to be clean, read it — do not scroll past it.

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
    "isolatedModules": true
  },
  "include": ["src"]
}
```

| Option                        | Why it is on                                                                                                      |
| ----------------------------- | ----------------------------------------------------------------------------------------------------------------- |
| `strict`                      | **Non-negotiable.** The entire point. Every "why is TypeScript annoying" complaint is a `strict: false` codebase. |
| `noUncheckedIndexedAccess`    | `arr[0]` returns `T \| undefined`. Your DOM queries and field arrays deserve that honesty.                         |
| `verbatimModuleSyntax`        | Forces `import type` for types. Faster builds, no runtime surprises.                                              |
| `moduleResolution: "bundler"` | How imports resolve in a Vite world.                                                                              |
| `skipLibCheck`                | Do not typecheck other people's code. Only yours.                                                                 |
| `include: ["src"]`            | **Which files to check.** This is why `hello.ts` lives in `src/`.                                                  |

> **`"include": ["src"]` is the quiet one.** Get this wrong and TypeScript silently checks
> nothing. Here is the real error, if you ever see it — it is confusing because it does not
> mention the file you meant to check:
>
> ```
> error TS18003: No inputs were found in config file 'tsconfig.json'.
> Specified 'include' paths were '["src"]' and 'exclude' paths were '[]'.
> ```
>
> That error means **your `src/` directory is empty or missing**, not that your code is fine.

> **Version warning.** Older tutorials show `"jsx": "react-jsx"` in the root config. From
> Chapter 3 onward this file gets split into shared presets in `packages/tsconfig/`. Do not
> build this file out by hand again after Chapter 2 — you will replace it with a preset.

---

## Step 3 — Git, and your first safety net

```bash
git add -A
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
git add .gitignore
git commit -m "chore: gitignore before any real code exists"
```

**Read that commit history.** In six months, when you are hunting a regression, this log is
the difference between a five-minute bisect and an afternoon. Commit messages are future
notes to yourself.

> **Your repo is not a git repo yet.** Check before you start:
>
> ```bash
> cd ~/projects/reprise && git status
> ```
>
> If you get `fatal: not a git repository`, then `refrain-buildmap/` and `refrain.md` are not
> under version control yet. **Fix that before Day 1**, because Chapter 2 creates `refrain/`
> as a separate repo and you will want the curriculum versioned too:
>
> ```bash
> cd ~/projects/reprise
> git init
> git add -A
> git commit -m "docs: the 19-chapter build map"
> ```

---

## Step 4 — Clean up

```bash
rm -rf ~/refrain-lab
```

You now have the toolchain and you understand every file in it. Chapter 2 starts the actual
monorepo.

---

## Your 60/40 split for this chapter

**Every task in this chapter, in the order it appears. 15 tasks: 15 yours, 0 delegated.**

| # | Task                                                    | Who | Why |
| - | ------------------------------------------------------- | --- | --- |
| 1 | `node -v`, `pnpm -v`, `git --version`                   | **You** | Three commands. Delegating this is delegating "is my machine set up" |
| 2 | `corepack enable` if pnpm is missing                    | **You** | It edits your shell config — a machine change, not a code change |
| 3 | Chrome `--version`, then read it against the 114/138 table | **You** | This decision determines Chapter 8's inference engine |
| 4 | `typeof LanguageModel` in DevTools                      | **You** | One line. The answer changes what you build later |
| 5 | Create the `refrain-dev` Chrome profile                 | **You** | A browser setting. Chapter 18's perf measurement depends on it |
| 6 | `mkdir -p ~/refrain-lab/src`, `git init`                | **You** | Includes the `src/` — the detail that silently breaks your typechecker |
| 7 | Write `src/hello.ts`                                    | **You** | 4 lines, and the whole point is the types, not the code |
| 8 | `pnpm add -D tsx`, then `pnpm exec tsx src/hello.ts`    | **You** | Local install, not global — you are learning why |
| 9 | **Break it on purpose**, read `TS2345`                  | **You** | ⚠️ **The actual lesson.** A test you read is worth more than one you delegate |
| 10 | Second experiment: `labels[0]`, produce `\| undefined`, then toggle the flag off and compare | **You** | Seeing what the flag *catches* is how it becomes instinct |
| 11 | Restore `hello.ts`, write `package.json` from scratch  | **You** | It is 15 lines and it teaches what every field means |
| 12 | Write `tsconfig.json` from memory                       | **You** | ⚠️ **Most important single task in Chapter 1.** See below |
| 13 | Two `git` commits + `.gitignore`                       | **You** | The commit history is your only regression tool later |
| 14 | `git init` the docs repo if it isn't one               | **You** | Protects the curriculum itself |
| 15 | **Explain every `tsconfig` line out loud**              | **You** | ⚠️ **The real deliverable of Chapter 1** |

> **Chapter 1 delegates nothing, and that is correct — not an oversight.** There is no
> boilerplate here to hand off. Every single task is either a decision about how strictly you
> want to be told you are wrong, or an experiment whose *output you must read yourself*.
>
> **The reason is worth naming: every option in `tsconfig.json` is a rule about your own
> future code.** If you paste that file without reading it, you have installed twelve rules you
> cannot evaluate when one of them blocks you in Chapter 9 — and your instinct will be to turn
> `strict` off rather than fix the code. Reading it once here is what prevents that.

### What OpenCode *is* allowed to do in this chapter

Not implementation — **understanding help.** Use it for these, and nothing else:

| Ask it | Why this is safe to delegate |
| ------ | ---------------------------- |
| *"Quiz me on what each `tsconfig` option does. Ask one at a time. I will answer; do not correct me until I say go."* | It cannot write your understanding for you. It only checks it. |
| *"I got `error TS2345`. Explain what TS2345 means and what part of my code it points at. Do not fix it."* | Reading an error is slower than being told the answer, and that slowness is the lesson |
| *"What is the difference between `moduleResolution: bundler` and `node`, and when would I want the other?"* | Reference knowledge, not a decision you are making today |
| *"Explain what pnpm's content-addressable store does, in plain language."* | Understanding, not configuration |

> **The one rule for this chapter: OpenCode may explain and may quiz, but it may not write a
> single line you commit.** If it produces a file, you have skipped the only thing Chapter 1
> has to teach.

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

**`error TS18003: No inputs were found in config file`.** Your `src/` directory is empty or
missing, or your file is at the root instead of inside `src/`. It does **not** mean your code
is fine.

**Your editor is red everywhere after `strict: true`.** Expected — that is `strict` earning its
keep. Do not turn it off to make the red go away. Fix the code.

**`tsc` prints nothing and you think it did not run.** It ran and found nothing. Silence is
success. Confirm it is actually checking by introducing a deliberate type error — if that
produces no output, your `include` is wrong.

**`tsx` works but `pnpm start` says `command not found`.** You installed `tsx` globally but
declared it locally, or you are running the script before `pnpm install`. Check
`package.json` → `devDependencies`.

---

## Verify before moving on

You are done with Chapter 1 when **all** of these are true:

- [ ] `node -v` and `pnpm -v` report the versions above
- [ ] You know your Chrome version and whether you have the `LanguageModel` API
- [ ] You can explain what `type: "module"` does without looking it up
- [ ] You can explain every line of `tsconfig.json` out loud
- [ ] You have read a TypeScript error and understood it without copying the fix
- [ ] You have produced the `| undefined` error and can explain why the flag caused it
- [ ] You have a `.gitignore` in a repo with two commits in it

---

## Check yourself before Chapter 2

If you cannot answer these, reread the chapter. They will come up constantly.

1. **Why can't a web app fill a form on another website?** Name the rule.
2. **What is the only component allowed to touch the page, and what is it forbidden from doing?**
3. **You open a form and the extension reports zero fields. What is your first hypothesis?**
4. **What does `noUncheckedIndexedAccess` do, and why does your content script need it?**
5. **What breaks if you put an extension in your personal Chrome profile?**
6. **Three programs ship but there are two apps. Why is that not a contradiction?**

---

**Next: [Chapter 2 — The Monorepo](./02-monorepo.md)** — pnpm workspaces, Turborepo,
`apps/` and `packages/`, and why `pnpm dev` will boot three programs at once from two apps.