# 02 — TypeScript that matters

> **Read this after Chapter 2.** You will have written the Zod schemas by then,
> and this explains what those types are actually doing for you.

---

## 1. Structural, not nominal

TypeScript does **not** use class inheritance for types. It uses **structural
typing**: two types are compatible if they have the same shape.

```ts
type UserId = { id: string }
type DocId = { id: string }

// These are DIFFERENT types to a human, and IDENTICAL to TypeScript.
declare const a: UserId
declare const b: DocId
a = b   // no error — structurally identical
```

**Why this matters enormously for Refrain:** the whole point of
`packages/fields` is that the extension and the API agree on a wire format
**without importing each other's code.** Structural typing is what makes that
possible — both sides validate against the same *shape*, and there is no
inheritance hierarchy to keep in sync.

**The flip side, and it is a real trap:** TypeScript will not stop you passing a
`DocId` where a `UserId` is expected. For that you need a branded type:

```ts
type Brand<T, B> = T & { readonly __brand: B }
type UserId = Brand<string, "UserId">
type DocId = Brand<string, "DocId">

declare const d: DocId
function takeUser(id: UserId) {}
takeUser(d)   // ✗ error — the brands do not match
```

**Use brands for IDs.** `docId` and `userId` are both strings, and swapping
them produces a bug that looks like missing data rather than a type error.

---

## 2. Narrowing, and why you need it

TypeScript cannot always know what you have, so you help it.

```ts
// Discriminated union — the single most useful TS pattern
type State =
  | { kind: "idle" }
  | { kind: "fermata"; ready: number; needs: number }
  | { kind: "attacca"; count: number }

function render(s: State) {
  if (s.kind === "idle") {
    // s is { kind: "idle" } — no ready, no count. Verified.
    return "Ready when you are"
  }
  if (s.kind === "fermata") {
    // s.ready is number. TypeScript KNOWS.
    return `${s.ready} ready, ${s.needs} need you`
  }
  return `${s.count} filled`
}
```

`kind` is the **discriminant**. Every branch narrows, and accessing a property
that does not exist in that branch is a compile error.

**This is what `packages/fields` is doing.** `MappingResult.status` is a
discriminated union, so Chapter 8's resolver cannot forget to handle
`needs-input` — the compiler will not let it.

### The `never` exhaustiveness check

```ts
function assertNever(x: never): never {
  throw new Error(`Unhandled: ${JSON.stringify(x)}`)
}

function render(s: State) {
  if (s.kind === "idle") return "Ready"
  if (s.kind === "fermata") return `${s.ready} ready`
  // Missing the attacca branch? This line errors.
  return assertNever(s)
}
```

**Add a fifth state and TypeScript tells you every place you forgot.** That is
why `packages/ui`'s mascot uses exactly this — it is the difference between a
new state that works everywhere and one that silently falls through.

---

## 3. `unknown` vs `any` — never interchangeable

```ts
const a: any = userInput
a.whatever.nested.thing()   // ✅ compiles. This is the disaster.

const b: unknown = userInput
b.whatever                   // ✗ error — you must check first
```

**`any` disables the compiler. `unknown` disables *you* until you check.**

Refrain parses hostile input constantly — the DOM, uploaded PDFs, extracted
text. Every such boundary uses `unknown`:

```ts
const raw: unknown = JSON.parse(text)
const schema = ProfileSchema.safeParse(raw)   // narrows unknown → Profile
if (!schema.success) return { error: schema.error }
profile = schema.data                          // now typed
```

> **Interview line:** "`unknown` is the safe default for anything crossing a
> boundary. `any` is a deliberate escape hatch that I have to justify — and in
> this codebase the only justification is `crypto.subtle`'s typings, which are
> incomplete."

---

## 4. Generics — the type equivalent of a function parameter

A generic takes a type as an argument.

```ts
// Without generics, you either lose the type or cast everywhere
function first<T>(items: T[]): T | undefined {
  return items[0]
}

const name = first(["a", "b"])        // string | undefined  ✓ inferred
const age = first([1, 2])             // number | undefined  ✓ inferred
```

### `extends` is a constraint

```ts
// Constrain to keys that actually exist
function get<T extends object, K extends keyof T>(obj: T, key: K): T[K] {
  return obj[key]
}

declare const user: { id: string; age: number }
get(user, "age")        // number  ✓
get(user, "nickname")   // ✗ error — "nickname" is not a key of user
```

**`T[K]` is an indexed access type** — the type of `obj[key]`. That is how you
get a precise return type without a cast.

**Where Refrain uses it:** the mapping engine. A resolver is generic over the
value type it produces, so `resolveField` returns `MappingResult` with the right
inner type whether it produced a string or a number.

---

## 5. Zod: runtime validation as types

TypeScript types **vanish at runtime**. This is the thing people miss.

```ts
type User = { id: string }
// Compiles to NOTHING. This line does not exist in the built output.
const x: User = JSON.parse('{"id": 42}')  // a lie, and TS believes it
```

**Zod fixes this.** You define the shape once, and get both a type and a
validator that runs at runtime:

```ts
import { z } from "zod"

const UserSchema = z.object({
  id: z.string(),
  age: z.number().int().min(0),
})
type User = z.infer<typeof UserSchema>   // type from the schema

// At runtime, this ACTUALLY CHECKS
const parsed = UserSchema.safeParse(input)   // input: unknown
if (!parsed.success) return { error: parsed.error }
user = parsed.data                            // correctly typed
```

**This is the single most important thing to say about Refrain's architecture:**

> "Types are erased. Everything crossing a boundary — the DOM, a PDF, an API
> response — is `unknown` at compile time and gets validated by Zod at runtime.
> That is why `packages/fields` is a *runtime* module with a type re-export, and
> not an interface file. An interface would have produced a `syncmeta` row with
> no content guard at all."

### `.strict()` vs default stripping — the subtle one

```ts
const A = z.object({ id: z.string() })
const B = z.object({ id: z.string() }).strict()

A.safeParse({ id: "1", value: 87.4 })   // ✅ SUCCESS — `value` silently STRIPPED
B.safeParse({ id: "1", value: 87.4 })   // ✗ FAILS — unrecognized key
```

**Default Zod is lenient, and that is normally correct** — it is what lets an
old client talk to a new server. But for a *field* object, an unrecognised key
is always a typo, so `FormFieldSchema` uses `.strict()`.

Chapter 2's test caught this by accident: the original test asserted
`success === false` and passed for four unrelated reasons.

---

## 6. `satisfies` — the operator you want more of

```ts
// ❌ `: T` widens. Loses literal types. Everything becomes string.
const config: Config = { name: "Refrain", port: 3000 }

// ❌ no annotation — no checking at all
const config = { name: "Refrain", port: 3000 }

// ✅ `satisfies` checks the shape AND keeps the narrow type
const config = {
  name: "Refrain",
  port: 3000,
} satisfies Config
// config.port is 3000 (literal), not number
```

Used in `turbo.json`'s tasks and in the `MASCOT_PARAMS` table: it proves the
table matches `MascotParams` while keeping each state's literal values narrow.

---

## 7. Compiler flags that actually matter

From `tsconfig.base.json`, and *why* each is on:

| Flag | Why |
|---|---|
| `strict` | Non-negotiable. Every "TS is annoying" complaint is a `strict: false` codebase. |
| `noUncheckedIndexedAccess` | `arr[0]` is `T \| undefined`. Your DOM queries deserve that honesty. |
| `verbatimModuleSyntax` | Forces `import type`. No runtime import of a type you did not need. |
| `noImplicitOverride` | Catches a missing `override` when extending a class. |
| `noFallthroughCasesInSwitch` | Catches a missing `break`. **This is why the mascot reducer is exhaustive.** |
| `skipLibCheck` | Do not typecheck other people's code. Only yours. |

**`noUncheckedIndexedAccess` deserves its own note**, because it is the one
people disable first:

```ts
const fields = schema.fields
const first = fields[0]        // FieldSchema | undefined

first.label          // ✗ error, correctly — this array could be empty
```

**Your content script reads `document.querySelectorAll(...)` results constantly.**
Without this flag, `nodes[0].getAttribute("id")` compiles and throws at runtime
on an empty NodeList — which is exactly the "extension reports zero fields" bug
in Chapter 7.

---

## 8. Questions you must be able to answer

1. Structural vs nominal typing. Give an example of each.
2. Why do branded ID types exist? What bug do they prevent?
3. What is a discriminated union, and what does it give you?
4. Explain the `assertNever` exhaustiveness check and when you would add one.
5. `any` vs `unknown`. Which is the default, and why?
6. **Why do TypeScript types not exist at runtime?** What breaks because of it?
7. What does Zod add that an interface cannot?
8. Default Zod strips unknown keys. When is that right, and when is it a bug?
9. What does `satisfies` do that `: Type` does not?
10. Why is `noUncheckedIndexedAccess` on, and which bug in this codebase does
    it prevent?
11. Which of your compiler flags would you turn off if you were in a hurry, and
    what would you break?

---

*Next: [03 — HTTP, and why extensions can](./03-http-and-the-browser.md)*