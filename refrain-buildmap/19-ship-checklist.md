# Chapter 19 — Ship Checklist

> **Day 23 · Goal: the extension is submitted, the privacy policy is verifiable, and you know
> exactly what you did not build.**
>
> The final gate. Everything before this chapter made the product work. This one makes it
> trustworthy — and trustworthy is the whole product.

---

## Understand this first

### You are asking a stranger for a credential

Not a password. Not a bank login. But a device-wide permission to **read every form you fill
and write into them.**

A Chrome permission prompt says: *"Refrain: Read and change your data on forms.google.com,
naukri.com, and 3 other sites."* A reasonable person deciding whether to install should have
enough information to say yes. **Everything in this chapter exists to give them that.**

That reframes the rest of the work. The code is done or it is not. What remains is making the
product legible to someone who cannot read your source and will not read it if you give them
reason not to.

### The reviewer is the same person as your user, with four minutes

A Web Store reviewer with `<all_urls>` permission in front of them is making the exact decision
your user makes: *is this developer reading all my data?* **You are not being evaluated by a
specialist. You are being evaluated by your actual user, under time pressure, with incomplete
information.**

That means:

| Reviewer question | What answers it |
|---|---|
| "Does it read all my data?" | A short, specific `host_permissions` list |
| "Does it send my data anywhere?" | A privacy tab that says "no", plus a verifiable grep |
| "Does it submit forms for me?" | A limitation explicitly stated in the listing |
| "Why do I need this permission?" | The "Why these sites" field, answered properly |
| "Is the developer real?" | An open repo, a real README, a person |

**A rejection is a documentation bug.** Nearly every Chrome Web Store rejection for a
privacy-focused extension is because the listing did not answer a question the reviewer had. The
code was fine.

### Two checkpoints, fifteen days in

| Day | Checkpoint | Proves |
|---|---|---|
| **8** | **"I found 14 fields."** Scan a real portal, show provenance on every one | The core is real |
| **13** | **One form filled end to end.** Review → correct → fill → you press submit | It actually works |
| **15** | Submitted to the store, privacy policy live | It is shippable |

**Day 8's checkpoint is the honest one.** "I found 14 fields" is achievable with the content
script and the mapping engine alone. If you cannot clear it, nothing later matters. **Do not
skip to Day 13 and hope.** A product that scans perfectly and cannot fill is a product that
demonstrates failure, and you want that failure on Day 8 while you can still fix it.

---

## Step 1 — The privacy policy that is verifiable

Most privacy policies are aspirational. **This one is checkable, and the checkability is the
point.**

```markdown
<!-- PRIVACY.md -->
# Refrain — Privacy

**Last updated: 2026-10-XX**

## The short version

Refrain works with no account and no network connection.

Everything you enter stays on your device, encrypted with a key
derived from your passphrase. Nothing is sent to us unless you turn
on sync, and even then we store ciphertext we cannot read.

## What stays on your device

Your profile (name, email, phone, address, CGPA, and anything else you
add), every document you upload, every saved answer, and your
history of prepared forms. All encrypted with AES-256-GCM, all in
your browser's local storage.

## The one exception: optional sync

If you create an account and turn sync on, we store:

- Your email address (needed to identify your account)
- Encrypted blobs of your data — we hold the ciphertext and the
  key-wrapping material. We cannot decrypt either. We have no
  server-side key that can open your data.
- Metadata needed to make sync work: a document id, a revision
  number, a timestamp, and a byte count.

We do **not** store field names, field values, document filenames,
or document contents in any readable form.

If you never turn sync on, your data never reaches our servers at
all. The extension makes no network requests.

## What we collect

Nothing. No analytics. No telemetry. No error tracking in the
extension. No crash reports containing your data.

Our API server records request metadata (timestamp, route, status
code, a redacted device label) for operational purposes. Field
values, field labels, document names, and email addresses are
excluded from logs by design.

## Third parties

None. There is no analytics service, no advertising network, no
error-reporting SDK, and no cloud AI provider in this product.
Inference runs entirely on your device.

## What your extension can see

Refrain's content script runs on the pages listed in its
permissions, and only those pages:

- forms.google.com
- naukri.com
- internshala.com
- (+ the specific portals listed in our store listing)

On those pages it reads form fields to offer to fill them. It does
not read page content otherwise. It does not transmit anything.

## Verifying these claims

Every claim on this page is checkable:

    grep -rniE '2captcha|selenium|puppeteer|webdriver' packages/ apps/
    grep -rniE 'google-analytics|gtag|mixpanel|amplitude|sentry' packages/ apps/
    grep -rn 'fetch(' packages/vault/src/

These return nothing. Our repository is public, our build is
reproducible, and our privacy gates run on every pull request.

## Deletion

Delete your account in Settings. Access is revoked immediately and
all server data is permanently deleted within 30 days.

To delete what is only on your device, use "Delete vault" in
Settings, or clear the extension's storage in your browser.

## Contact

you@refrain.dev · github.com/refrain
```

**Five things make that page credible and four of them are structural:**

**It states what we store when sync is on, in detail.** Vagueness here is what makes people
suspicious, and here the details are *reassuring* — a reader who expected field names and got
"a revision number and a byte count" leaves more confident than one who got a paragraph of
assurances.

**It has a verification section with runnable commands.** This is the single highest-leverage
thing on the page. It converts a claim into a check, and it costs four lines.

**It names the specific domains.** "It only runs on 5 sites" is checkable. "Only where needed" is
not.

**It says what our logs exclude.** Nobody asks about logs. The fact that you addressed it
unprompted is the signal.

**It is honest about the sync exception.** §11 originally said "no server." Chapter 12 rewrote
that. **A privacy policy that is now accurate is worth more than one that was impressive and
wrong**, because the wrong one ends with someone who checked.

---

## Step 2 — The Web Store submission

### Listing copy

```markdown
**Name (45 chars max):**
Refrain — answer once, fill everywhere

**Short summary (132 chars max):**
Fill any web form from your own encrypted profile. Local-first, open source, and you always press submit.

**Category:**
Productivity

**Language:**
English
```

### The description — the part that gets read

```markdown
Refrain fills web forms using information you entered once, on your
own device. It does not submit anything. You always press submit.

WHAT IT DOES
• Scans the page for form fields
• Shows you which values it found, and where each one came from
• Fills only what you approve
• Never submits. Never solves CAPTCHAs. Never runs on CAPTCHA pages.

WHY THESE SITES
Refrain's permissions are limited to the form portals we support.
We do not request access to all websites. If you see Refrain on a
site outside this list, something is wrong — please report it.

WHY THIS IS SAFE
• Your profile is encrypted with AES-256-GCM, on your device
• Nothing is sent to a server. There is no account required
• Every value is shown with its source before it is written
• Refrain never submits a form. There is no code path that can.
• The source is open. You can grep it.

NO ACCOUNTS, NO CLOUD, NO AI API
Refrain runs inference on your own machine (Chrome's built-in AI, or
Ollama). Your data is never sent to a model provider.

WHAT IT DOES NOT DO
• No form submission
• No CAPTCHA solving
• No bulk form filling
• No analytics or telemetry
• No advertising

FREE AND OPEN SOURCE
MIT licensed. github.com/refrain

Works offline. Sync across devices is optional and, if enabled,
end-to-end encrypted.
```

**Six decisions in that copy, and each one exists for a reason:**

| Choice | Why |
|---|---|
| "You always press submit" in the **first two lines** | The single most important trust statement, above the fold |
| "WHY THESE SITES" as a named section | Directly answers the reviewer's first question |
| "If you see Refrain on a site outside this list, something is wrong — report it" | Turns a limitation into an integrity signal |
| "No code path that can" | States it as a fact about the code, not a promise |
| "The source is open. You can grep it" | Invites verification. Reviewers like it |
| "WHAT IT DOES NOT DO" | Reviewers look for a limitations section. Having one is a strong signal |

> **"You always press submit" belongs in the short summary, not just the description.** The
> short summary is the only text visible without clicking. Whatever claim you most want a user
> to read before deciding, it goes there.
>
> **The limitations section is unusual and it works.** Every other listing writes about what the
> product does. A section listing what it refuses to do reads as confident, because it implies
> someone thought about the boundaries. §11 is the product — making it visible is marketing.

### Screenshots

| # | Shows | Why it exists |
|---|---|---|
| 1 | Side panel open on a real Google Form, 14 fields found, mascot in `fermata` | The core value in one image |
| 2 | Provenance chips close up — "from a document", "you typed this" | **The differentiator.** Nobody else has this |
| 3 | The marksheet drop zone → extracted fields with a locator | Shows the hardest feature works |
| 4 | The Setlist with Encore | The retention story |
| 5 | Mascot's four states in a row | Design credibility |
| 6 | Settings → "Nothing sent" | The privacy claim, visually |

```bash
# 1280×800 or 640×400. Chrome does the scaling.
# Real screenshots only. Never a Figma mockup with invented data.
# A screenshot of real UI with a real (redacted) form is
# worth more than a beautiful one with placeholder text.
```

> **Screenshot 2 is your best asset and most developers will not make it.** The provenance chips
> are the one feature in §12's competitive table that nobody else has. A screenshot showing "from
> a marksheet, p.3" next to a value makes the trust argument in one image, with no marketing
> required.
>
> **Use real forms with real (redacted or fake) data.** A screenshot of a Google Form with
> lorem ipsum in it reads as a prototype. One with a real-looking form reads as a product.

### Privacy practices — the form that decides approval

```
[X] I do not collect or transmit user data
[ ] I collect or transmit user data — details:
```

**This is the critical checkbox and you must be precise about it.**

| Field | Your answer | Why |
|---|---|---|
| Data collection | **None** from the extension | Extension makes no requests |
| Data use for the user's benefit | — | N/A |
| Data sold to third parties | **None** | |
| Data used for advertising | **None** | No ads exist |
| Remote code execution | **No** | §11 no-list, verified in CI |
| Collection of web browsing activity | **No** | Content script does not report pages |
| Cryptography use | **Yes — encryption of user data on device** | Declare it honestly |

> **If sync is shipped, this form gets more complicated.** Once you have an API holding
> ciphertext, you are transmitting user data (encrypted) and must declare it. The accurate
> answer is "encrypted data, plus email address and sync metadata" with the promise that it is
> never read. **Do not check the "no data" box while shipping an account system** — that is
> misrepresentation, it is discoverable, and it ends a Chrome publisher account permanently.
>
> The pragmatic sequencing: **submit Phase 1 with no sync** (which is a complete product), then
> add sync and update the disclosure in a later version. Chapter 12 made sync optional for
> exactly this reason.

### Common rejection reasons

| Rejection | Cause | Fix |
|---|---|---|
| "Remote code execution" | Fetching Tesseract WASM from a CDN | Bundle it. §18 A2 |
| "Permissions not justified" | `<all_urls>` with no explanation | Surgical list + the WHY THESE SITES section |
| "Privacy disclosure mismatch" | Ships sync but checked "no data" | Declare it accurately |
| "Single purpose" | Listing describes many features | Lead with one sentence. One purpose |
| "Deceptive" | Screenshots that do not match the product | Real screenshots only |
| "Data not encrypted in transit" | Any HTTP endpoint | HTTPS everywhere, HSTS preloaded |

> **Review takes 1–14 days and you cannot rush it.** Submit in week one with a minimal build if
> you have to — a working v0.9 that is in review while you finish is strictly better than a
> finished v1.0 that starts its clock on day fifteen.

---

## Step 3 — The README that converts

Your README is the landing page. It is also what the reviewer skims.

```markdown
# Refrain

**Answer once. Fill everywhere.**

A Chrome extension that fills web forms from information you entered
once — stored encrypted on your own device.

It never submits. You always press submit.

<!-- The screenshot must be first. Above every badge. -->
![Refrain](docs/screenshot.png)

## Why

Filling a scholarship form should not take forty minutes because you
have typed your address four hundred times. Refrain remembers, and
shows you where every value came from before it writes anything.

## How it works

1. **Profile** — enter your details once, or drop in a marksheet
2. **Scan** — open any form; the side panel finds the fields
3. **Review** — every value shown with its source and confidence
4. **Fill** — you approve, Refrain fills
5. **Submit** — **you do this. Refrain has no code path that can.**

## The trust model

Everything is encrypted on your device with AES-256-GCM, using a key
derived from your passphrase with PBKDF2 (600,000 iterations).

No account. No server. No telemetry. No cloud AI. Inference runs on
your machine — Chrome's built-in model, or Ollama.

You can verify every claim:

    grep -rniE '2captcha|selenium|puppeteer|webdriver' packages/ apps/
    grep -rniE 'google-analytics|gtag|mixpanel|amplitude' packages/ apps/

Both return nothing.

## Architecture

    packages/fields    Zod contracts. The one definition of a field.
    packages/vault     AES-GCM, PBKDF2, Dexie. All encryption.
    packages/ui        React components, design tokens, the mascot.
    packages/mapping   Label normalisation, aliases, rules.
    packages/extract   pdf.js + Tesseract. Document extraction.

    apps/extension     Chrome MV3. Side panel + content script.
    apps/web           The editor. React 19 + Vite.
    apps/api           Optional sync. Hono + MongoDB.

## Status

Phase 1 · MIT licensed · Chrome Web Store

## Contributing

The mapping alias dataset (`aliases.json`) is the moat. Contributions
welcome — see docs/ALIASES.md for the labelling format.
```

**Six things that make it convert:**

**A screenshot above every badge.** Nobody decides to install a project from a row of build
badges.

**"It never submits. You always press submit." in the second line.** The trust claim is above
the fold, because that is what the reader needs before anything else.

**The greps, in the README, runnable.** Not a privacy promise. A copy-pasteable check. This is
the most unusual thing in the README and it is the most likely reason someone installs.

**The architecture in seven lines.** It shows there is a design, not a script. Contributors and
reviewers both read this section.

**A `Status` line that says Phase 1.** Honesty about maturity prevents the wrong expectations.

**A named contribution target.** `aliases.json` is the moat and it is 100% hand-labelled work
(Chapter 8). Saying so converts a contributor into a labelling partner, which is the only kind
of contribution that scales.

---

## Step 4 — Phase 1 exit criteria

**You are done when every line is checked. Not "mostly."**

### The core loop

- [ ] Enter 20+ facts via `/profile`
- [ ] Scan a real portal → **14+ fields found**, correct labels
- [ ] Every mapped value shows a provenance chip with a reason
- [ ] Low-confidence values are amber and require a tap
- [ ] Fill works on a text input
- [ ] Fill works on a select dropdown
- [ ] Fill works on a radio group
- [ ] Fill works on a date input
- [ ] Fill works inside an **iframe**
- [ ] Fill works on a **hidden-input widget** (findHiddenDriver)
- [ ] Fill works on a **React-controlled** input (native setter)
- [ ] Review → correct → fill → **you press submit**
- [ ] Refrain does not submit. Ever.

### The three bugs, explicitly

The bugs from Chapter 7 produce identical symptoms and different causes. **Test all three
separately or you will think you passed.**

- [ ] **Bug 1** — React-controlled input: value is visible AND the form receives it
- [ ] **Bug 2** — iframe form: fields found with `allFrames` + `frameId` on write
- [ ] **Bug 3** — hidden driver: the *stored* value updates, not the visible decoy

```bash
# Bug 1 verification — the visible text is not the proof.
# This is the check that matters:
node -e '
  const el = document.querySelector("input[name=\"email\"]");
  const setter = Object.getOwnPropertyDescriptor(HTMLInputElement.prototype, "value").set;
  setter.call(el, "test@example.com");
  el.dispatchEvent(new Event("input", { bubbles: true }));
  console.log("element value:", el.value);
  // Now read the React state, not the DOM. If you only check
  // el.value, you will "pass" this test while shipping Bug 1.
'
```

### Documents

- [ ] A text PDF extracts correct, correctly-ordered lines
- [ ] A scanned PDF routes to OCR automatically
- [ ] OCR confidence is 0.62 and **no OCR value ever auto-fills**
- [ ] Every fact has a locator with page number and matched line
- [ ] Clicking the chip opens the source at that page
- [ ] Confirming promotes to `manual` at 1.0
- [ ] A promoted fact fills with no warning on a later form
- [ ] A **sideways photo** is rotated and OCRs correctly

### Privacy — the greps

```bash
# Every one of these must return nothing.
grep -rniE '2captcha|anticaptcha|capsolver|deathbycaptcha' packages/ apps/
grep -rniE 'selenium|puppeteer|playwright|webdriver' packages/ apps/ --include='*.ts' --include='*.tsx' | grep -v test
grep -rniE '\.click\(\)|requestSubmit\(|\.submit\(\)' packages/ apps/ | grep -v test
grep -rniE 'google-analytics|gtag|mixpanel|amplitude|posthog|segment\.com' packages/ apps/
grep -rniE 'sentry|bugsnag|datadog' apps/extension/
grep -rE 'https?://' apps/extension/.output/chrome-mv3/manifest.json
grep -rn 'console.log' packages/vault/src/crypto/
git ls-files | grep -E '\.env|\.pem$'
```

- [ ] All eight return nothing
- [ ] The privacy gates run in CI and fail the build
- [ ] `PRIVACY.md` exists and every claim in it is true
- [ ] The §11 blocked-page list refuses correctly and says why

### Vault

- [ ] PBKDF2 at 600,000 iterations
- [ ] `k_vault` is `extractable: false`
- [ ] A wrong passphrase cannot decrypt anything
- [ ] Lock clears the key **and** the UI updates
- [ ] A crash mid-write leaves the vault openable (Dexie version not renumbered)
- [ ] Export produces a re-importable archive
- [ ] Delete clears everything, in the right order

### The two metrics that decide whether this worked

```ts
// Add these to Settings. They are the product's own quality measure.
const stats = {
  /** Fields Refrain filled that the user then corrected. */
  correctionRate: corrections / totalFilled,
  /** Facts answered once and never needed again. */
  repeatAnswerRate: factsReused / factsEntered,
}
```

- [ ] `correctionRate` is measured and **decreasing** over time
- [ ] `correctionRate` under 10% on a typical form
- [ ] `repeatAnswerRate` above 70% after 20 forms

> **`correctionRate` is the single number that tells you if this product is working.** Every
> other metric is a proxy. A 4% correction rate means Refrain's mapping is right 96% of the
> time; a 30% rate means the alias dataset needs work and no amount of UI polish will fix it.
>
> **And it must be shown to the user.** Chapter 10 puts "2 corrected" quietly in the Setlist row.
> That is not a metric display — it is an integrity signal, and a user who sees that number
> trusts you more precisely *because* it admits you are imperfect.

---

## Step 5 — The release

```bash
# 1. Version every package coherently.
pnpm -r version minor

# 2. The full gate. No exceptions, no "it built last time."
pnpm check

# 3. The privacy gates.
bash scripts/privacy-gate.sh

# 4. Manual smoke test, on the real forms you care about.
#    Not automated. You. With your eyes.
```

```
The manual smoke test:
  [ ] Google Form — text, select, radio, date
  [ ] Google Form inside an iframe
  [ ] A React-heavy portal (Naukri)
  [ ] A portal with a custom dropdown widget
  [ ] Marksheet upload → extract → confirm → fill
  [ ] CAPTCHA page → refuses, explains, does not act
  [ ] A page with 50+ fields → panel stays responsive
  [ ] Keyboard-only pass through the review screen
  [ ] Dark mode on every screen
  [ ] Vault: setup, lock, unlock, wrong passphrase, export, delete
```

```bash
# 5. Tag it. The tag is the version.
git tag ext-v1.0.0
git push origin main --tags

# 6. The workflow builds, verifies, and uploads to the store.
#    Watch it. Do not walk away.
```

**Ship it small.** v1.0.0 with the core loop, profile, documents, and the Setlist. **Not** with
sync, not with Verses, not with accounts.

```bash
# Defer to v1.1:
#   - Optional sync (Chapters 12-16)
#   - Verses with live compression preview (Chapter 10)
#   - Ollama fallback
```

> **Why defer sync despite building eight chapters of it.** Because a Web Store extension
> that requires an account is a smaller product with a smaller audience, and because sync is the
> one thing that makes your §11 claim complicated. **Ship the version that needs no server,
> prove people want it, then add sync to the people who asked for it.** The backend is not
> wasted work — it is waiting, fully built, for a demand signal.
>
> **That is a real discipline and it is worth naming:** eight chapters of backend code that
> does not ship in v1.0 is not waste, it is a bet that the demand exists. Betting on your own
> roadmap before anyone has used the product is how you end up with a sync engine and no users.

---

## Step 6 — What Refrain does not do

The most important section in this entire document. Put it in the README, the store listing,
and hold it to it.

| Refrain does not | Why |
|---|---|
| **Submit forms** | The line the product is built around. `filled` has no `SUBMIT` transition |
| **Solve CAPTCHAs** | §11. Automated solving is how tools become abuse tools |
| **Bypass detection** | Same reason. A tool that evades bot detection is a tool for abuse |
| **Bulk fill** | "Fill 200 forms" is the same capability as spam |
| **Run on every site** | Permission is surgical, and it is a trust feature |
| **Send your data anywhere** | No model API, no analytics, no error reporting in the extension |
| **Work without you** | Every fill requires a review. There is no unattended mode |
| **Answer for you** | A motivation essay is written by you. Refrain only resizes it |
| **Guess** | Below 0.75 it asks. There is no "probably fine" path |

> **Every one of these is a competitor's feature.** Refrain's entire differentiation is
> *declining* to do eight things that are technically easy and collectively corrosive. A product
> that does not do the corrosive things has a reason to exist that a product that does them
> cannot have.
>
> **And this list is your defence when someone asks why you will not add a feature.** "Why no
> auto-submit?" — it is the line. "Why no CAPTCHA solving?" — it is the line. "Why not run
> everywhere?" — it is the line. **A product with explicit non-goals makes decisions in minutes
> that products without them spend weeks on.**

### The features you will be asked for, and the answer to each

| Request | Answer |
|---|---|
| "Auto-submit for forms I've already filled" | No. Ever. That is the product |
| "Fill 50 internship applications at once" | No. That is spam with a UI |
| "Support more sites automatically" | The alias dataset grows by hand. That is the moat |
| "Sync to my phone automatically" | Yes — in v1.1, opt-in, end-to-end encrypted |
| "Read my WhatsApp messages" | No. §11. There is a WhatsApp section precisely to say no |
| "Let me share my profile with my college" | No institutional tier in Phase 1, and a real compliance problem |
| "Work offline completely, no server ever" | Already true. That is v1.0 |

---

## Step 7 — After you ship

### The first week

```
Day 1-2   Read every piece of feedback. Do not fix anything.
Day 3     Triage: bug, UX, or feature? Fix bugs. Nothing else.
Day 4-5   Fix the bugs. Add the alias entries the feedback implies.
Day 6-7   Ship v1.0.1.
```

> **Do not fix anything in the first 48 hours.** Early feedback is dominated by setup errors —
> the passphrase did not work, the panel did not open, they did not understand the review screen.
> **Those are not bugs to fix; they are copy to rewrite.** Reacting to day-one feedback produces
> churn against problems you do not yet understand.

### The three things that will actually break

```ts
// 1. A portal redesign. Your labels still match; the ids changed.
//    → Encore's label fallback (Ch.10) saves you. Add the aliases.
// 2. A form framework doing something you have never seen.
//    → findHiddenDriver misses a new widget pattern. Teach it.
// 3. A user who genuinely needs auto-submit.
//    → They do exist. They are not your user, and serving them
//      turns you into a tool they would rather you were not.
```

**All three are content problems, not code problems.** They need the alias dataset and the
hidden-driver library to grow — which is exactly why both are designed to be extended by a
human labelling, not a model generating guesses.

### Where the time goes next

| Work | Why | Who |
|---|---|---|
| **Alias labelling** | The moat. 300 entries now; the best portals need 2,000 | **You, forever** |
| Real-form regression tests | Every portal you add is a test fixture | You + OpenCode |
| Portal-specific drivers | Each new widget is new work | You |
| Community PRs on `aliases.json` | The only contribution that scales | Community |

> **The alias dataset is the moat and it is the only part of Refrain that cannot be
> automated.** Every competitor can copy your encryption — it is WebCrypto. Every competitor can
> copy your UI. **Nobody can copy 2,000 hand-verified label mappings for Indian scholarship
> portals**, because labelling them requires sitting with real forms and knowing which of two
> similar labels is the one that means what it says.
>
> That is the actual business. Everything else is table stakes.

---

## Commit

```bash
git add -A
git commit -m "docs: PRIVACY.md, README, Web Store listing, Phase 1 exit criteria, non-goals"
```

Add one row to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | v1.0 ships without sync, despite the backend being built | An extension that requires an account is a smaller product with a smaller audience, and sync is the one thing that complicates the §11 claim. Prove demand with the no-server version. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The whole privacy policy** | **You, 100%.** Every claim is a legal assertion you are making |
| **Verifying every claim in it is true** | **You.** Then `grep` again |
| **The "what Refrain does not do" list** | **You. 100%.** This is the product's identity |
| **The answers to the seven feature requests** | **You** |
| **Why defer sync to v1.1** | **You** |
| **Which three things will break** | **You** |
| **The store listing copy** | **You** |
| **The screenshots** | **You** — real UI, real data |
| **The README** | **You** |
| The manual smoke test | **You.** All twelve lines, with your eyes |
| The triage process for week one | **You** |
| Running the greps | **OpenCode** — but **read each result yourself** |

---

## Gotchas in this chapter

**Rejected: "remote code execution."** Tesseract's WASM fetched from a CDN. Chapter 18 A2. MV3 forbids it.

**Rejected: "permissions not justified."** `<all_urls>` in the shipped manifest. Chapter 18 C2.

**Rejected: "privacy disclosure mismatch."** You ship sync and checked "no data." Declare it.

**Rejected: "single purpose."** The description reads as a feature list. Lead with one sentence.

**A screenshot looks like a prototype.** Lorem ipsum in a real form. Use a real form.

**The review takes 13 days.** It does. Submit in week one with a minimal build.

**The passphrase does not work on Windows and you cannot reproduce it.** Chrome's
`crypto.subtle` on a non-secure origin — you are on `http://localhost` and it works, but a
packaged app served over `file://` does not have `crypto.subtle`. **This is the single most
common "works for me" bug in an extension** and it only appears after packaging.

**The panel does not open in the store version.** `sidePanel.default_path` does not match your
built path, or `host_permissions` does not cover the site.

**The Chrome Web Store listing says "sync is available" and it is not.** Do not describe
features you did not ship.

**You shipped with a `.env` committed.** `git ls-files | grep env`. Chapter 17.

**`correctionRate` is 30% and you shipped anyway.** The mapping is wrong and no UI will fix it.
Go work on aliases.

**A user asked for auto-submit and you said yes.** That is the line. Chapter 19 Step 6.

**You are reading day-one feedback as feature requests.** It is mostly setup confusion. Wait 48
hours.

---

## Verify before you ship

- [ ] All Phase 1 exit criteria checked, not "mostly"
- [ ] All three content-script bugs tested **individually**
- [ ] Bug 1 verified against React **state**, not the DOM
- [ ] All eight privacy greps return nothing, in CI and locally
- [ ] `PRIVACY.md` exists and every claim is verifiable
- [ ] The store listing's short summary contains "you always press submit"
- [ ] The description has WHY THESE SITES and WHAT IT DOES NOT DO
- [ ] Screenshots are real UI, 1280×800 or 640×400, at least one of provenance chips
- [ ] The privacy practices form is accurate
- [ ] README has the screenshot first and runnable greps
- [ ] The manual smoke test is complete, all twelve lines
- [ ] The manual smoke test was run on the **packaged** build, not just `pnpm dev`
- [ ] `correctionRate` is measured and under 10%
- [ ] Version tagged and the store upload succeeded
- [ ] You can name all eight things Refrain does not do
- [ ] You know which three things will break first

---

## Check yourself before you start building

1. **Why submit a minimal build in week one instead of the finished product on day 15?**
2. **Why does the privacy policy include runnable greps?**
3. **Why is "WHY THESE SITES" a named section in the listing?**
4. **Why must you not check "no data collected" while shipping sync?**
5. **What is the difference between verifying Bug 1 against the DOM and against React state?**
6. **Why is `correctionRate` the metric that decides whether this worked?**
7. **Why ship v1.0 without sync when eight chapters of backend are already built?**
8. **Why is the alias dataset the moat rather than the encryption?**
9. **Why wait 48 hours before acting on feedback?**
10. **What is the value of a product with explicit non-goals?**

---

**That is the whole map.** Nineteen chapters, from `pnpm init` to a submitted extension.

You now have:
- A 7-package monorepo with enforced boundaries
- A design system and a mascot with four states
- AES-256-GCM encryption with a non-extractable key
- A content script that survives React, iframes, and hidden drivers
- A mapping engine with rules beating a model 92% of the time
- A review screen where the human gate is structural
- A document pipeline that is safe for OCR
- A setlist that gives users a reason to return
- A MongoDB backend with hybrid storage and honest sync
- CI, deployment, accessibility, and a privacy policy you can grep

**And a list of eight things you refuse to do.**

That last list is the product. Everything else is how you deliver it.