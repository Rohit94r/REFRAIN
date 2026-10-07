# 01 — JavaScript that matters

> **Read this before Chapter 1.** Everything here shows up in code you have
> already written. If any of it is unfamiliar, that is the gap — not a
> preference.

---

## 1. The module system, and why `type: "module"` matters

JavaScript has two module systems and mixing them fails at runtime.

```js
// CommonJS — old. `require`, `module.exports`
const express = require("express")

// ES Modules — modern. `import` / `export`. This is what you use.
import express from "express"
```

Node decides which one based on **`package.json`**, not on file extension:

```jsonc
{
  "type": "module"   // every .js file in this package is ESM
}
```

**The failure worth knowing:** a `.js` file with `import` in a package without
`"type": "module"` fails with `Cannot use import statement outside a module`.
The error is accurate and unhelpful, because the fix is in `package.json`, not
in that file.

`.mjs` always means ESM. `.cjs` always means CommonJS. Those are the escape
hatches when you cannot change `package.json`.

---

## 2. `let`, `const`, and the loop-closure bug

`var` is function-scoped and hoisted. `let` and `const` are block-scoped.

```js
for (var i = 0; i < 3; i++) setTimeout(() => console.log(i))
// 3, 3, 3  ← one binding, mutated after the loop

for (let i = 0; i < 3; i++) setTimeout(() => console.log(i))
// 0, 1, 2  ← a fresh binding per iteration
```

**Why it matters:** `let` gives each loop iteration its own variable. That is
the entire mechanism, and it is why `var` produces a genuinely different
program rather than a stylistic difference.

**Rule:** `const` by default, `let` when you must reassign, `var` never. You
already have `strict: true`, but that is about types, not this.

---

## 3. Closures — the one that gets asked every time

A closure is a function that remembers where it was created.

```ts
function makeCounter() {
  let count = 0              // this survives after makeCounter returns
  return () => ++count
}

const c = makeCounter()
c()  // 1
c()  // 2
c()  // 3

const d = makeCounter()     // a completely separate counter
d()  // 1                    ← not 4
```

**Where Refrain uses this:** the vault session. `VaultSession` holds the derived
key in a private field with a `#` prefix, and `unlock()` returns functions that
close over it. The key is not a module-level variable that anything can
reach — only the returned methods can.

```ts
class VaultSession {
  #key: CryptoKey            // # = private to this class, not just by convention

  async unlock(passphrase: string) {
    this.#key = await deriveKey(passphrase)
    // These close over `this`, so the key stays reachable only here.
    return { read: () => this.read(), lock: () => this.lock() }
  }
}
```

> **Interview framing:** "A closure captures its lexical environment by
> reference, not by value. So mutations inside the closure are visible
> afterwards, and two closures created by separate calls share nothing."

---

## 4. `this` — four rules, in priority order

`this` is not where the function is **defined**. It is determined by **how it is
called**.

```js
// 1. new → a fresh object
new Thing()        // this = the new instance

// 2. Method call → the object before the dot
obj.method()       // this = obj

// 3. Plain call → undefined (strict mode) or globalThis (sloppy)
plain()            // this = undefined

// 4. Arrow functions have NO this. They inherit from the enclosing scope.
```

The arrow-function rule is the one that bites:

```js
const obj = {
  name: "Refrain",
  regular() { return this.name },        // "Refrain"
  arrow: () => this.name,                // undefined — `this` is module scope
}
```

**So arrow functions cannot be object methods.** But they are ideal inside a
component or callback, because they inherit the right `this`:

```tsx
// ❌ class field with an arrow — works, but creates a per-instance function
class Button {
  onClick = () => this.track()
}

// ✅ prototype method — one function shared by all instances
class Button {
  onClick() { this.track() }
}
```

That is a memory optimisation, not a correctness one. Both are correct.

---

## 5. The event loop — where async actually lives

JavaScript runs on **one thread**. Async does not add threads; it schedules
callbacks for later.

```
Call stack  →  running right now
Microtasks  →  promise callbacks. Run to completion before anything else.
Macrotasks  →  setTimeout, I/O callbacks. Run after microtasks drain.
```

```js
console.log("1")                     // sync — now

setTimeout(() => console.log("4"))  // macrotask

Promise.resolve().then(() => console.log("3"))  // microtask

console.log("2")                     // sync — now

// Output: 1, 2, 3, 4
// Not 1, 2, 4, 3 — microtasks always drain before the next macrotask.
```

**The rule that matters in this codebase:** `await` yields to the microtask
queue, so a long loop of awaits still blocks rendering if it is tight. That is
why Chapter 18 puts `setTimeout(r, 0)` between frame batches in the content
script — not to be polite, but to actually return control to the browser so it
can paint.

> **Interview version:** "Node has an event loop with a libuv threadpool for
> I/O. The browser has one thread for JS and a separate process for layout
> and paint. An `await` lets the event loop run other callbacks, but it does
> not create parallelism — CPU-heavy work still blocks, which is why OCR in
> Refrain runs in a Web Worker."

---

## 6. Promises and the three mistakes

```ts
// 1. Forgetting await — a silent, catastrophic bug
const data = fetchProfile()      // a Promise, not data
console.log(data.name)           // undefined, no error

// 2. Unhandled rejection — the app dies silently
button.addEventListener("click", () => deleteVault())   // returns a Promise nobody awaits

// 3. Sequential await when the work is independent
const user = await getUser()     // ❌ 300ms
const settings = await getSettings()  // ❌ another 300ms
// ✅ parallel — 300ms total
const [user, settings] = await Promise.all([getUser(), getSettings()])
```

**Refrain is full of independent work.** The content script needs the page
schema, the stored profile, and the alias table. Those should be
`Promise.all`, not a chain — and Chapter 18's 250ms scan budget is mostly
spent if they are sequential.

**`Promise.all` vs `Promise.allSettled`:** `all` rejects on the first failure
and you lose the other results. `allSettled` returns every outcome. For the
content script you want `allSettled` — one failed DOM query should not discard
the profile you already loaded.

---

## 7. Immutability, and why it is not a style choice

```ts
const facts = [{ key: "name", value: "A" }]

// Mutation — changes the SAME object
facts[0]!.value = "B"

// Spread — creates a new array AND a new object at [0]
const updated = facts.map((f) => (f.key === "name" ? { ...f, value: "B" } : f))
```

**Why Refrain cares:** React decides whether to re-render by comparing object
identity. If you mutate in place, React sees the same reference and does not
re-render — so a corrected value silently does not appear on screen.

```tsx
// ❌ React does not re-render. The row keeps the old value.
fact.value = newValue

// ✅ New identity, React re-renders
setFacts(facts.map((f) => (f.key === fact.key ? { ...f, value: newValue } : f)))
```

This is the single most common React bug, and it looks identical to "the state
update did not work." Always ask about identity before you debug the setter.

---

## 8. `Map`, `Set`, and object keys

```ts
const m = new Map<string, number>()
m.get("missing")      // undefined, NOT a crash — unlike obj.missing on a plain object

const s = new Set(["profile", "document"])
s.has("profile")      // true
```

Use `Map`/`Set` rather than `{}` and `[]` when keys are dynamic, you insert
frequently, or you iterate — plain objects are slower for these and their keys
are strings only.

In `packages/vault`, the profile graph uses a `Record<string, ProfileFact>`
because the keys are known at compile time (TypeScript gives you autocomplete).
That is the right choice *there*. The distinction is: **known keys → object,
dynamic keys → `Map`.**

---

## 9. Questions you must be able to answer

1. What does `"type": "module"` change, and what error do you get without it?
2. Why does `var` in a loop produce `3, 3, 3`?
3. What is a closure? Give an example that is not a counter.
4. What does an arrow function do to `this`, and where can you not use one?
5. Microtasks vs macrotasks. What is the output order, and why?
6. What is the difference between `Promise.all` and `allSettled`, and when do
   you need the latter?
7. Why does mutating an object break a React re-render?
8. `Map` or object for a profile graph with 40 known keys? Why?
9. Why does OCR in Refrain run in a Web Worker?
10. Where in this codebase is there CPU-bound work, and how does that risk
    blocking the event loop?

---

*Next: [02 — TypeScript that matters](./02-typescript-that-matters.md)*