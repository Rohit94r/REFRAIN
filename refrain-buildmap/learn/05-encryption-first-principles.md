# 05 — Encryption, from first principles

> **Read this after Chapter 6, not before.** You will have hit an IV collision
> and a tamper-detection failure by then, and this will explain why those were
> not arbitrary.
>
> **This is the single most important document in this folder.** Refrain's
> entire product promise is one cryptographic claim, and you are about to own
> it.

---

## 1. Hashing is not encryption

The most common confusion in the interview, and in real code.

| | Hashing | Encryption |
|---|---|---|
| Direction | One-way. You cannot reverse it. | Two-way. Decrypt with a key. |
| Purpose | Verify integrity, lookup by fingerprint | Recover the original |
| Key needed? | **No** | **Yes** |
| Changes input | Changes output completely | Tiny change → tiny change in output |

```ts
// Hashing: same input, same output, always. No key.
const hash = await crypto.subtle.digest("SHA-256", data)
// You cannot get `data` back. Ever.

// Encryption: reversible, but only with the key.
const key = await crypto.subtle.generateKey(...)  // AES-GCM
const ct = await crypto.subtle.encrypt({ name: "AES-GCM", iv }, key, data)
const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct)  // only works with key
```

**Why this matters for Refrain:** your passphrase is **never** stored, and it
is **never** hashed into something reversible. It is used as *key material* to
derive an encryption key. If you had hashed the passphrase and stored the hash
as an "auth token," anyone with the hash could log in — which is exactly the
"forgot password" problem Chapter 14 forbids by architecture.

---

## 2. The key hierarchy — why one key is not enough

The beginner design: one passphrase → one AES key → encrypt everything.

**It is wrong**, and the reason is blast radius. Every document shares one
key, so:

- You cannot rotate the passphrase without re-encrypting every document
- You cannot share access to one document
- A bug in key handling exposes *everything*

Refrain uses four layers:

```
passphrase
    │  PBKDF2-SHA256, 600,000 iterations, salt = HMAC(email)
    ▼
k_root                     ← never leaves the device
    │  HKDF-SHA256, two purposes
    ├──────────────► k_vault  ──► wraps per-document DEKs   ← NEVER SENT
    └──────────────► k_auth   ──► sent to server, Argon2id-hashed there
```

**Why `k_vault` is never sent:** the server must be able to authenticate you
without being able to decrypt your data. So there are two keys derived from
one root. The server gets `k_auth`. You keep `k_vault`. That is the entire
mechanism of end-to-end encryption in one sentence.

**Why each document gets its own DEK (data encryption key):**

```
k_vault ──wraps──► DEK_document1 ──encrypts──► fact 1..40
         ──wraps──► DEK_document2 ──encrypts──► fact 41..80
```

This means changing your passphrase only **re-wraps** the DEKs — it does not
re-encrypt any data. Re-encrypting 40 documents of 4MB each would take minutes
and could fail halfway. Re-wrapping 40 tiny keys takes milliseconds and is
atomic.

> **This is the single most useful thing to say in an interview about this
> system.** "Rotation re-wraps keys, it does not re-encrypt data, which is
> why it is safe to do online." Anyone who has built key management says yes.

---

## 3. AES-GCM, and what GCM means

AES alone is a **block cipher**. It encrypts 16-byte blocks. On its own it is
deterministic: the same plaintext with the same key always gives the same
ciphertext.

That is fatal for your use case. Two identical facts — the same city in two
address fields — would produce identical ciphertext. An attacker with database
access could count occurrences and see which values repeat. Worse, they could
replay an old ciphertext.

**GCM (Galois/Counter Mode)** wraps AES and adds three things:

| | What it adds |
|---|---|
| **Nonce / IV** | A unique number per encryption. Same key + different nonce → completely different ciphertext. |
| **Authentication tag** | A MAC over the ciphertext. Decryption **fails loudly** if anyone changed a byte. |
| **Associated data (AAD)** | Extra bytes authenticated but not encrypted. |

So AES-GCM gives you confidentiality **and** integrity in one operation.

---

## 4. The IV bug you will hit

This is the single most common crypto mistake, and Chapter 6 makes you hit it
on purpose.

```ts
// ❌ BROKEN — and it passes every functional test
const iv = new Uint8Array(12)   // all zeros
const ct = await encrypt(data, key, iv)
```

**Why it is catastrophic, not just bad:**

GCM's security reduces to the IV never repeating under the same key. Reuse
breaks it *completely*, not partially:

1. XORing two ciphertexts produced with the same key and IV **reveals the XOR
   of the two plaintexts.** `C1 ⊕ C2 = P1 ⊕ P2`.
2. **The authentication key is recoverable.** With two messages you can solve
   for the GHASH subkey, and then forge a valid tag for *any* data.

So an attacker with two ciphertexts gets meaningful plaintext, and can
manufacture ciphertext your own code will accept.

**The rule: a fresh, random, unique IV for every single encryption.**

```ts
// ✅ Correct
const iv = crypto.getRandomValues(new Uint8Array(12))  // 96 bits, random
```

Not a counter you increment. Not a timestamp. **Random.** A counter is fine
only if you can guarantee uniqueness across restarts and crashes — which is
harder than it sounds, and is why we do not.

> **Why 12 bytes / 96 bits:** it is GCM's native size, and Chrome's WebCrypto
> only accepts 12 bytes for AES-GCM. Other libraries allow other sizes. This
> is one of the rare cases where the platform's limitation is also the right
> choice.

**The test that catches it:** encrypt the same plaintext twice, assert the two
ciphertexts differ. That single assertion is in `packages/vault`'s test suite
for this reason.

---

## 5. Password hashing vs key derivation — a real distinction

Different problems, and Refrain uses both. On purpose.

| | Purpose | Algorithm | Salt | Stretched? |
|---|---|---|---|---|
| **KDF** | Turn a low-entropy passphrase into a key | PBKDF2 / Argon2id / scrypt | Yes | **600,000 iterations** |
| **Password hash** | Verify a login without storing the password | Argon2id | Yes | Yes, plus memory-hard |

Both are deliberately **slow**. That sounds like a bug and it is the entire
point — it is what makes guessing expensive for an attacker.

**PBKDF2 vs Argon2id, and why Refrain uses both:**

- **`k_vault`** is derived with **PBKDF2**, on the client. It must run inside a
  browser `crypto.subtle`, and WebCrypto ships PBKDF2 — not Argon2id.
- **The server's `k_auth`** is verified with **Argon2id**, because the server
  can install a native module. Argon2id is memory-hard, which is strictly
  better against GPU attacks.

**So: PBKDF2 because the platform gives us nothing better. Argon2id because
where we can choose, we choose better.** That is a defensible answer and it is
the honest one.

### Why 600,000 iterations and not fewer

PBKDF2 is not memory-hard, so attackers use GPUs — billions of guesses per
second. You fight that with iteration count.

The cost lands on **you**: roughly 250ms on a laptop, closer to 900ms on a
low-end Android phone. Chapter 18 says do not lower it, and show a progress
state instead.

> **The reasoning to give in an interview:** "PBKDF2 is not memory-hard, so I
> raised the iteration count to compensate. I kept it because the alternative —
> weakening the KDF to make unlock feel fast — makes offline guessing cheaper.
> I showed progress instead."

---

## 6. Tamper detection — the part people forget

Encrypt-then-MAC is old advice. With GCM you do not do it separately — the tag
is built in, and **you must check it.**

```ts
// ❌ Decrypt without checking the tag
const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct)
// Attacker swapped ct for an older valid ciphertext. You cannot tell.

// ✅ Correct — WebCrypto checks the tag internally and THROWS
try {
  const pt = await crypto.subtle.decrypt({ name: "AES-GCM", iv }, key, ct)
} catch {
  throw new Error("Vault integrity check failed")
}
```

The critical subtlety: **WebCrypto rejects a bad tag before returning any
plaintext.** That is why decrypt-then-verify in two steps is wrong — you must
let the operation fail atomically, never decrypt to a buffer and check later.

> **The interview question:** "What happens if an attacker with database access
> modifies a fact?" Answer: GCM's tag covers the ciphertext, so decryption
> throws and the document is rejected. They cannot alter one field without
> detection. **Then the follow-up:** "What if they delete it?" Answer: that is
> a different attack, handled by tombstones in Chapter 13, because
> authentication cannot detect absence.

That second answer is what separates someone who has built it from someone who
has read about it.

---

## 7. Questions you must be able to answer

Say them out loud. Then read the answer, not before.

1. What is the difference between hashing and encryption? Why does Refrain
   never store the passphrase?
2. Draw the key hierarchy. What never leaves the device?
3. Why two keys from one root instead of one key?
4. Why does each document get its own DEK?
5. What does the G in AES-GCM add over plain AES?
6. What happens if you reuse an IV with the same key? Be specific about what
   the attacker gets.
7. Why is the IV 12 bytes?
8. Why random IVs and not a counter?
9. PBKDF2 or Argon2id for the server? Why does the client use the weaker one?
10. Why 600,000 iterations, and what is the cost?
11. An attacker with database access modifies one fact. What happens?
12. An attacker deletes one fact. What happens, and why is that a different
    problem?

---

*Or go back to the [fundamentals index](./README.md).*

*Next: [the fundamentals index](./README.md). Document 06 on Databases is
written the day you reach Chapter 13.*