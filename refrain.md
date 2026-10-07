# Refrain

> **Your personal AI form companion.**
> **Answer once. Fill everywhere.**

| | |
|---|---|
| **Product** | Refrain |
| **Domain** | `refrain.layerflow.dev` |
| **Status** | Pre-build · design locked · nothing written yet |
| **Document** | Master document — everything in one file |
| **Current phase target** | **Phase 1 (MVP)** — one Google Form filled end-to-end |
| **Last updated** | 2026-10-05 |

---

## Contents

1. [Name & Brand](#1-name--brand)
2. [The Problem](#2-the-problem)
3. [The Product](#3-the-product)
4. [What It Is Not](#4-what-it-is-not)
5. [Audiences](#5-audiences)
6. [Architecture](#6-architecture)
7. [The Six Pillars](#7-the-six-pillars)
8. [Surfaces](#8-surfaces)
9. [Design & UX](#9-design--ux)
10. [Tech Stack](#10-tech-stack)
11. [Trust, Privacy & Compliance](#11-trust-privacy--compliance)
12. [Competitive Landscape & Moat](#12-competitive-landscape--moat)
13. [Business Model](#13-business-model)
14. [Roadmap](#14-roadmap)
15. [Success Metrics](#15-success-metrics)
16. [Long-Term Vision](#16-long-term-vision)
17. [Operating Principles](#17-operating-principles)
18. [Open Questions & Risks](#18-open-questions--risks)
19. [Decisions Made](#19-decisions-made)
20. [Build Order](#20-build-order)
21. [Homepage Copy](#21-homepage-copy)
22. [Shareable Brief](#22-shareable-brief)
23. [One-Line Summary](#23-one-line-summary)

### The build curriculum

This document is **what** to build. The companion folder
[`refrain-buildmap/`](./refrain-buildmap/README.md) is **how and in what order** — the same
product broken into eight chapters across fifteen days, from "install Node" to a packaged
Chrome extension. Read a chapter before you build that part, not after.

| # | Chapter | Days | Deliverable |
|---|---|---|---|
| 1 | [Foundations](./refrain-buildmap/01-foundations.md) | 1 | A TypeScript project you can explain line by line |
| 2 | [The Monorepo](./refrain-buildmap/02-monorepo.md) | 2 | `pnpm dev` boots all three surfaces |
| 3 | [Design System & Mascot](./refrain-buildmap/03-design-system-and-mascot.md) | 3 | Four mascot states, provenance chips designed |
| 4 | [The Vault](./refrain-buildmap/04-the-vault.md) | 4–5 | Encrypted profile with provenance on every fact |
| 5 | [The Content Script](./refrain-buildmap/05-the-content-script.md) | 6–8 | **"I found 14 fields."** ← demo checkpoint |
| 6 | [The Mapping Engine](./refrain-buildmap/06-the-mapping-engine.md) | 9–10 | Every field mapped, rule-based, with confidence |
| 7 | [Fermata](./refrain-buildmap/07-fermata-review-screen.md) | 11–12 | **One form filled end-to-end** ← MVP |
| 8 | [The Setlist, Documents & Ship](./refrain-buildmap/08-setlist-documents-and-ship.md) | 13–15 | Tracker live, extraction working, package built |

Where the build map deliberately departs from this document, it says so inline and asks you to
record the decision here in §19. Those are the only places the two can disagree.

---

## 1. Name & Brand

### The name: **Refrain**

A *refrain* is **the part of a song that keeps coming back.** That is the entire product —
your information, returning every time it is asked for.

**It is also a double meaning, deliberately.** *Refrain from* — to hold back. The product never
auto-submits. **The hardest constraint in §11 is inside the word.**

**Why this name:**

- 7 letters, two clean beats. No `th`, no silent letters, no consonant clusters. It survives
  being said once on a college WhatsApp call and typed once in a search bar.
- Sounds like a **tool**, not a toy. Works equally well for a 19-year-old and a CFO.
- **Not descriptive — which is the point.** It does not box the product into "student forms"
  on the day the positioning expands to job seekers and professionals. A name like *FillOnce*
  or *OneFill* would.
- **Distinctive is a security feature.** A stranger must search it, and searching leads to the
  GitHub repo, the audit, and the privacy policy. A generic name gets pattern-matched to the
  spam extensions students are already warned about. §11 is the go-to-market — the name has to
  spend that budget well.
- **Clean of funded software companies.** Verified against 16 candidates before adoption; this
  was the only one with no meaningful collision. See §19.

### Brand word system

One musical family. Every word below is a real term of music, and costs nothing to register.

| Word | Refers to |
|---|---|
| **Refrain** | The product |
| **Verse** | A saved reusable answer — *"that's my verse about myself"* |
| **Variation** | A Verse adapted to a specific form's question |
| **Motif** | A fact that recurs across every form — your college, in 20 applications |
| **Setlist** | The Application Tracker — the ordered list of everything you've played |
| **Encore** | One-tap re-fill of a form you have already completed |
| **Tempo** | The rhythm of your application cycle / recurring forms |
| **Fermata** | **The human gate.** The hold, where you decide. Named because it is non-negotiable |
| **Attacca** | Go straight on without pausing — what happens after you press Submit |

**Why this family holds:** a Refrain only means something because it **returns.** So the whole
product is built on recurrence — your Verse comes back as a Variation, listed on your Setlist,
at your Tempo, every time.

**Never name a sub-brand after a mechanism.** *OneFill Docs, OneFill Pro, OneFill Tracker* —
that is a feature label, not a brand. This family is nouns. They stay nouns as the product
grows.

### Tagline

> **Answer once. Fill everywhere.**

**The split matters:** the *name* is the thing you are loyal to. The *tagline* is the promise.
`OneFill` tries to be both, and ends up competing with Chrome's AutoFill for the weaker half.

### Positioning sentence

> Do **not** say *"AI form autofill."*
> Say **"your personal AI form companion."**
> Then: **Answer once. Fill everywhere.**

---

## 2. The Problem

People repeatedly type the same information into different forms. One person may do this
hundreds of times a year.

```
Name · Email · Phone · DOB · Address · College · Degree · Branch · Semester
10th marks · 12th marks · CGPA · Skills · Experience · Projects · Achievements
Resume · Certificates · Portfolio · Long-form answers
```

### The real problem is rephrasing

Forms ask for the same thing in different words:

```
Full Name
Candidate Name
Applicant's Name
Name of Student
Name as per marksheet
Enter your name
```

Conventional autofill matches **exact labels**. Miss the label, the field stays blank, and the
user fills it manually. **That is the entire gap in the market.**

### Why existing tools do not close it

| Tool | Why it fails here |
|---|---|
| **Password managers** (1Password, Bitwarden) | String match on labels. No semantic understanding. No document extraction. |
| **Browser automation** (Browse AI, Axiom.ai) | Built for recurring business workflows. Priced per runtime-hour. Requires the user to *build* an automation. Wrong shape for a one-off form. |
| **Browser AI agents** (Strawberry, Comet) | General-purpose, enterprise-priced, cloud-based, credit-metered. |
| **Native Chrome/Edge AI autofill** | Generic. Not document-derived. Not portal-aware. No application memory. |

**Nobody** has solved **document-derived data** — reading "87.4%" out of a marksheet PDF and
putting it in the right box.

---

## 3. The Product

### In one sentence

> A personal AI form and application companion that lets anyone save their information once
> and intelligently reuse it across forms, applications, registrations, scholarships,
> internships, jobs, and other repetitive online workflows.

### The full architecture

```
              ┌──────────┬──────────────┬──────────┐
              │   WEB    │  EXTENSION   │ WHATSAPP │
              │   APP    │   (action)   │(converse)│
              └────┬─────┴───────┬──────┴────┬─────┘
                   │             │           │
                   └─────────────┼───────────┘
                                 │
                        PERSONAL PROFILE
                                 │
              ┌──────────────────┼──────────────────┐
              │                  │                  │
       Personal Info      Document Vault     Reusable Answers
              │                  │                  │
              └──────────────────┼──────────────────┘
                                 │
                          AI UNDERSTANDING
                                 │
                  ┌──────────────┴──────────────┐
                  │                             │
            FORM READER                   SMART MAPPING
                  │                             │
                  └──────────────┬──────────────┘
                                 │
                           AUTO-FILLING
                                 │
                              REVIEW  ← human gate
                                 │
                        USER CONFIRMATION
                                 │
                       APPLICATION TRACKER ("The Setlist")
                                 │
              ┌──────────────┬────┴───────┬──────────────┐
           STUDENTS      JOB SEEKERS   PROFESSIONALS    EVERYONE
```

### The user experience, end to end

```
1.  User saves their profile once — facts, documents, answers
2.  User lands on any repetitive form
3.  Companion reports: "I found 21 fields. 18 ready. 3 need you."
4.  User clicks Fill
5.  Review screen shows every field + where the value came from
6.  User fixes what's missing
7.  User presses Submit            ← always the human
8.  Companion logs it to the Setlist
9.  Next form: Encore. One tap.
```

---

## 4. What It Is Not

Non-negotiable. Each of these is a different company, and every one is a trap.

- ❌ A browser
- ❌ A fully autonomous agent
- ❌ An RPA platform
- ❌ Institution / college management software or ERP
- ❌ A job board or scholarship marketplace
- ❌ A social network
- ❌ A huge CRM
- ❌ An AI chatbot with 100 unrelated features
- ❌ CAPTCHA bypass
- ❌ Automatic submission without human confirmation

**One central problem, everything supports it:**

> Repeatedly providing the same personal information online.

---

## 5. Audiences

### Primary — Students *(starting audience, testing ground)*

College forms · event registrations · hackathons · internships · scholarships ·
competitions · feedback forms · surveys · workshops · placement applications ·
clubs · fellowships · course registrations

### Second — Job seekers

Name, contact, education, experience, skills, projects, portfolio, resume, cover letters,
notice period, salary expectations, location, work authorization.
→ **your personal job-application companion**

### Third — Professionals

Freelancers · developers · designers · creators · researchers · entrepreneurs · employees ·
consultants · people applying for programs, certifications, events.

### Fourth — Everyone

Anyone tired of repeatedly typing their own information online.

### Positioning over time

| When | Positioning |
|---|---|
| Today | Built for students |
| Soon | Built for students and job seekers |
| Eventually | Built for everyone |

> **Students are the beachhead, not the ceiling.**

---

## 6. Architecture

### 6.1 The hard technical constraint

**A web app on your domain cannot touch forms on other websites.**

This is not a limitation to engineer around. It is the **same-origin policy**. A page at
`refrain.layerflow.dev` has zero access to the DOM of `forms.google.com` or any scholarship
portal. It cannot read inputs, set values, or click submit. Cross-origin iframes are blocked by
CSP and `X-Frame-Options`.

Only four things can read another site's form:

| # | Method | Verdict |
|---|---|---|
| 1 | Browser extension | ✅ **Chosen** — one-click install, Chrome Web Store distribution |
| 2 | Userscript (Tampermonkey) | ⚠️ Fallback — clunky UI, extra dependency |
| 3 | Bookmarklet | ⚠️ Fallback — no persistent UI, blocked by some CSPs |
| 4 | Native app (Electron / Selenium) | ❌ Two platforms, installer hell, massive scope |

Every product in this category is an extension or a browser **because this is the only option
that exists.** It is not a stylistic choice.

### 6.2 The resolution — proper web *is* compatible

The Chrome side panel (Chrome 114+) is **just a web page**. It can be a React + TypeScript
application with proper components, state, and routing.

So "proper web UI" and "can fill forms" are **not in conflict**. You need three surfaces
instead of one.

```
refrain.layerflow.dev   Full web app: onboarding, profile, documents, Setlist, settings
        ↕ shared encrypted vault (IndexedDB)
side panel              Companion UI — SAME React codebase, different entry point
        ↕ chrome.tabs.sendMessage
content script          The ONLY code allowed to touch the page. Dumb. No UI. No logic.
```

| Surface | Tech | Responsibility |
|---|---|---|
| Web app | React + TS (Vite) | Onboarding, profile, documents, answers, Setlist, settings, sharing |
| Side panel | Same React app, 2nd entry | Mascot, field detection, review screen, confidence |
| Content script | Vanilla TS | Read form schema. Write values. Nothing else. |

**Why the content script stays dumb:** it is the only component that can break when a portal
changes its HTML. Isolating it makes breakage a small, isolated, fixable diff.

### 6.3 Monorepo shape

```
refrain/
├── apps/
│   ├── web/           the full web app
│   ├── sidepanel/     companion UI (shares packages/ui)
│   └── extension/     MV3 shell + content script
└── packages/
    ├── vault/         encrypted IndexedDB profile + documents
    ├── mapping/       form schema → profile field resolution
    ├── extract/       pdf.js + Tesseract document parsing
    └── ui/            design system, mascot, review components
```

### 6.4 Message flow

```
User opens side panel on a form
        ↓
Side panel → content script: "extract form schema"
        ↓
Content script returns: [{label, name, type, required, options}, …]
        ↓
Side panel maps schema → profile (local model; labels only, never raw documents)
        ↓
Side panel renders REVIEW screen with per-field source attribution
        ↓
User approves → side panel → content script: "fill these values"
        ↓
Content script writes values (native setter + input/change events)
        ↓
CAPTCHA detected → side panel: "you solve this, I'll wait"
        ↓
User solves → user clicks SUBMIT (the human, always)
        ↓
Side panel logs the submission to the Setlist
```

### 6.5 Engineering gotchas that will cost you weeks if you don't know them

**Gotcha 1 — React-controlled inputs silently fail.**
Setting `input.value = "82.4"` makes the box *look* filled, but on submit the portal receives
an **empty string**, because React state never updated.

```js
const setter = Object.getOwnPropertyDescriptor(
  window.HTMLInputElement.prototype, 'value'
).set;
setter.call(el, value);
el.dispatchEvent(new Event('input',  { bubbles: true }));
el.dispatchEvent(new Event('change', { bubbles: true }));
```

Symptom if you skip it: *"works on Google Forms, every JSP portal submits blank."* The same
pattern applies to `<select>` (set `value`, dispatch `change`) and checkbox/radio groups
(set `checked`, dispatch `input` + `change`).

**Gotcha 2 — iframes.**
Many Indian portals render inside same-origin iframes. A content script without
`all_frames: true` sees a blank page and reports **zero fields**.

```json
{ "content_scripts": [{ "matches": ["<all_urls>"], "all_frames": true }] }
```

**Gotcha 3 — custom widgets.**
Jotform/Typeform and Indian portals often use hidden inputs driven by custom JS widgets.
Writing to the visible element does nothing if the widget's state lives elsewhere. Detection
must check for a paired `<input type="hidden">` and write there when one exists.

### 6.6 Site support strategy

Do **not** chase every website.

| Tier | Approach |
|---|---|
| **Tier 1** | Generic HTML forms + Google Forms — no site-specific code |
| **Tier 2** | Jotform, Typeform, Formspree, Microsoft Forms — generic adapters |
| **Tier 3** | Specific Indian portals — one small adapter per portal, only when users report it |

Every Tier 3 adapter is a maintenance liability. Add one only on demand.

---

## 7. The Six Pillars

### Pillar 1 — Personal Profile

Saved once. The user controls exactly what is stored.

```
Identity    Full name, email, phone, DOB, gender (where relevant)
Address     Street, city, state, pincode, country
Education   School, board, college, degree, branch, semester, grad year,
            marks, CGPA, subjects
Career      Skills, experience, projects, certifications, achievements,
            portfolio, GitHub, LinkedIn, languages, interests
```

**Design requirement:** the profile is **not** a rigid form. It is a flexible graph, because
real people hold overlapping facts — two degrees, three addresses, freelance plus salaried
work, a home address and a current address.

Every fact carries **provenance** — where it came from, and when. Provenance is what makes the
review screen trustworthy.

### Pillar 2 — Personal Document Vault

Reusable documents, **understood** — not merely stored.

```
📄 Resume  📄 CV  📄 Marksheets  📄 Certificates
📷 Profile Photo  ✍️ Signature  📄 Cover Letter  📄 Portfolio
```

The important part is **extraction, not storage**. A marksheet should yield:

```
Board       → Maharashtra State Board
Year        → 2024
Percentage  → 87.4%
Subject-wise → Maths 92, Science 88, English 81
```

Then when a form asks *"Percentage obtained in Class 10"*, the answer is already known.
**Upload once, reuse forever.**

**Extraction must be confidence-scored.** A scanned marksheet parsed by OCR is sometimes wrong.
Low confidence → surfaced for review, never silently filled. A confidently wrong marksheet
percentage is worse than an empty field.

### Pillar 3 — Reusable Answers

The long-form questions people rewrite endlessly:

```
Tell us about yourself.
Why do you want to join?
Why should we select you?
What are your career goals?
Describe your project.
What are your achievements?
```

**Blind pasting is not the product.** Refrain should understand the question and *adapt* the
stored answer — length, tone, and emphasis matched to what was actually asked.

A 40-word answer and a 250-word answer are different products for the same person. If the form
says *"in 50 words"*, respect it.

Naming: a saved answer is a **Verse**. An adapted one is a **Variation**.

### Pillar 4 — Smart Form Companion

The heart of the product. The user lands on a form:

> **18 fields found · 16 ready to fill · 2 need your attention**

| Form question | Refrain understands |
|---|---|
| Candidate Name | Full Name |
| Institution | College |
| Course | Degree |
| Academic Score | Relevant percentage / CGPA |
| GitHub Profile | GitHub |
| Resume | Resume document |
| About Yourself | Stored personal answer (Variation) |
| Percentage in Class 10 | Extracted from marksheet PDF |

**Pipeline:**

```
form detection → field extraction → schema normalisation
      → semantic mapping → confidence scoring → missing-info detection
      → fill → review → submit → log
```

### Pillar 5 — Review Before Submit

**A foundational principle, not a feature.**

```
✓ Name                Rohit Jadhav
✓ Email               ******@gmail.com
✓ College             Atharva University
✓ Degree              Electronics & Computer Science
✓ Resume              Rohit_Resume.pdf
✓ 10th Percentage     87.4%

⚠ Career goal                Needs review
⚠ Expected salary           Needs review

                              [ Submit ]
```

The message is **not** *"give an AI control over your identity."*
The message is **"let AI do the repetitive work. You make the final decision."**

Every field shows **where its value came from**:

| Marker | Meaning |
|---|---|
| ✅ | from your profile |
| ✅ | extracted from marksheet PDF |
| ✅ | your saved answer, adapted |
| ⚠️ | needs your input |
| 🔒 | low confidence — please verify |

### Pillar 6 — Application Tracker — "The Setlist"

Form filling is the funnel. **The Setlist is why users come back.**

```
Google Internship          ✓ Submitted    Oct 4
State Scholarship          ✓ Submitted    Oct 5
Hackathon XYZ              🟡 Interview   Oct 7
Company ABC                🔴 Needs docs  Oct 9
```

**Status states:** `Draft → Submitted → Interview / Rejected → Offer`, plus
**`Needs document`** — the state that causes real panic.

It grows into a searchable personal application history:

> *"Where did I apply last month?"*

Users should be able to ask the Setlist directly: what did I submit, what is pending, what is
overdue, what needs a document from my college.

**Encore:** once a form is completed once, re-filling it is **one tap**. This is the
compounding effect — the switching cost *is* the accumulated history.

---

## 8. Surfaces

### 8.1 Web App — the home

Simple and personal. **Not** an enterprise dashboard.

| Screen | Contents |
|---|---|
| **Home** | Time-aware greeting · counts · **what needs you today** |
| **Profile** | Full graph editor |
| **Documents** | Vault, extraction status, confidence |
| **Answers** | Saved Verses |
| **Setlist** | Application tracker, timeline, filters, deadlines |
| **History** | Every form encountered, what was filled, what was skipped |
| **Settings** | Privacy, encryption, integrations, account, export, delete |

### 8.2 Browser Extension — the action layer

The user never leaves the site they are on.

```
👋 I found 21 fields.
   18 ready to fill
   3 need your attention

   [ Fill ]
```

Then review, then submit. That is the whole interaction.

### 8.3 WhatsApp — the conversation layer

Not the main product. A way to manage information without opening a web app.

```
"My new phone number is +91XXXXXXXXXX"
"I changed my address."
[send resume PDF]
"Save this as my internship introduction."
"What information do you have about my education?"
"Remind me about the scholarship deadline."
```

**The experience should feel like:** your personal form assistant is always available.

⚠️ **Hard compliance gate before building this — see §11.4.**

### 8.4 Telegram — later

Same companion experience via a bot. Same underlying product. **Not in v1.**

---

## 9. Design & UX

### UX Principles

1. **The Review Screen is the hero screen.** Everything else is in service of it.
2. **The mascot is a state indicator, not a personality.** Four states only.
3. **Motion while working. Silence while the user types.**
4. **Always show provenance.** Where did this value come from?
5. **Never block the user.** Every automatic action is cancellable.
6. **Confidence is visible.** Low confidence is stated, not hidden.
7. **Let users correct the system** — every correction improves future fills.

### Mascot states

The four states are the musical family, in order: rest → listen → hold → go.

| State | Musical name | Display | Behaviour |
|---|---|---|---|
| **Idle** | **Rest** | "Ready when you are" | Quiet. No motion. |
| **Reading** | **Listening** | "Heard 14 fields on this page" | Subtle activity animation |
| **Needs you** | **Fermata** | "2 blanks, 1 attachment" | Attention state — calm, not alarming |
| **Done** | **Attacca** | "Filled — review and submit" | Brief positive state, then still |

**Fermata is the load-bearing one.** A fermata is a held note — a deliberate pause where the
performer waits. That is the human gate, and naming it means every surface that reaches it
says the same thing: *this will not move until you move it.*

**The test:** the mascot should make someone smile **once**, then become invisible. If it
narrates, celebrates, and greets, it fails. Character lives in the empty state, not the whole
interface.

### Design language

Reference *feel*: a companion panel beside the tab. **Not** their architecture.

- **Warm, rounded, friendly.** Duolingo, not Salesforce. The user is a person, not procurement.
- **One friendly geometric sans.** Warm accent colour, soft shadows, generous corner radius.
- **Full dark mode.**
- **Provenance chips are the visual signature** — the little ✅ / ⚠️ / 🔒 markers. Make this
  the most designed element in the product.

### Why mascot-driven UI is right here

Mascot-driven product UI is **culturally native to Indian consumer apps** — Duolingo,
PhysicsWallah, Zomato. Western tools that try it look gimmicky. In India it converts.

This is not decoration, it's a **distribution feature**: mascot UI is what students
screenshot and share. Your screenshot is your marketing.

**One honest caveat:** the mascot reads as friendly to students. If the institutional tier ever
gets serious, keep the mascot but make the *dashboard* restrained. Warm, not whimsical.

### Screens

**Side panel**

```
┌─────────────────────────────────┐
│  (•)  Refrain                    │
│                                 │
│  👋 I found 21 fields.          │
│                                 │
│  ✓ 18  ready to fill            │
│  ⚠ 3   need your attention      │
│                                 │
│  ─────────────────────────────  │
│  Name            Rohit Jadhav   │
│  Email           ******@gmail   │
│  10th %          87.4%  📄       │
│  College         Atharva Univ.  │
│                                 │
│  [ Fill 5 blanks ]  [ Fill all ] │
└─────────────────────────────────┘
```

**Review screen (the hero)**

```
┌──────────────────────────────────────────┐
│  Review before you submit                │
│                                          │
│  ✓  Name            Rohit Jadhav         │
│  ✓  Email           ******@gmail.com     │
│  ✓  College         Atharva University   │
│  ✓  Degree          Electronics & CS     │
│  ✓  Resume          Rohit_Resume.pdf     │
│  ✓  10th %          87.4%   📄 marksheet │
│                                          │
│  ⚠  Career goal     Needs your input     │
│     [ Tell me about yourself…         ]  │
│                                          │
│  ⚠  Expected CTC   Needs your input     │
│     [ ___                              ]  │
│                                          │
│         [ Back ]      [ Submit form ]    │
└──────────────────────────────────────────┘
```

Every row editable inline. No separate edit mode.

---

## 10. Tech Stack

### Monorepo

| Concern | Choice |
|---|---|
| Package manager | pnpm |
| Build orchestrator | Turborepo |
| Language | TypeScript everywhere |

### Web app & side panel (shared codebase)

| Concern | Choice | Note |
|---|---|---|
| Framework | React + Vite | Same app, two entry points |
| Styling | Tailwind CSS + shadcn/ui | Warm theme, one accent colour |
| State | Zustand + TanStack Query | Local-first, no server state by default |
| Forms | React Hook Form + Zod | Schemas shared with mapping engine |
| State machines | XState | Wizard navigation, conditional branches |
| Charts | Recharts | Setlist dashboard |

> **The mapping engine should share Zod schemas with the form layer.** One definition of
> "a field" used by both the UI and the resolver. Avoids a whole class of drift bugs.

### Extension

| Concern | Choice | Note |
|---|---|---|
| Manifest | Chrome Manifest V3 | Required |
| Build tool | Plasmo (or `@crxjs/vite-plugin`) | Either works; Plasmo has nicer side-panel DX |
| Content script | **Vanilla TS** | Never inject React into pages |
| Frames | `all_frames: true` | See §6.5 Gotcha 2 |
| Side panel | 2nd entry of shared React app | Chrome 114+ |
| Storage | IndexedDB via Dexie | Not `chrome.storage` — size limits |

### Local vault & encryption

| Concern | Choice |
|---|---|
| Storage | Dexie.js (IndexedDB) |
| Encryption | AES-GCM via WebCrypto |
| Key derivation | Argon2id / PBKDF2 from user passphrase |
| Session unlock | In-memory key, cleared on lock/timeout |

**The passphrase never leaves the device.** No recovery server, no backdoor — which is exactly
what makes the *"we can't see your data"* claim honest.

### Inference

| Tier | Engine | Use |
|---|---|---|
| Primary | Chrome built-in Prompt API (Gemini Nano) | Where available |
| Dev / fallback | Ollama (Llama 3.x / Qwen) | Local dev, older Chrome |
| Optional cloud | User-supplied API key | Only for genuinely hard mappings |

**Privacy rule:** mapping sends **field labels + a profile summary**. Raw documents and full
form contents never leave the device.

**Cost rule:** local-first means ~₹0 marginal cost per fill. This is the structural reason we
can serve students at all. Do not add a cloud call without a documented reason.

### Document parsing

| Concern | Choice |
|---|---|
| Text-layer PDFs | `pdf.js` |
| Scanned PDFs / images | Tesseract.js |
| Structured extraction | Local model, confidence-scored per field |

### Passport photo & signature

Indian forms frequently demand an exact-size passport photo and a scanned signature.

- Canvas capture (photo via camera, signature via touch/mouse)
- Auto-crop to the aspect ratio the form implies
- **Client-side resize before upload** — never round-trip a photo through a server

### Backend (thin, optional)

Cloudflare Workers / Hono:

- Accounts
- Licence keys
- Opt-in crash telemetry

**Stores no form content. No PII.** Verifiable in the public repository. If the backend ever
needs to handle form data, that is a design change requiring a new decision record.

---

## 11. Trust, Privacy & Compliance

### Operating principles

1. **User owns their data.**
2. **AI assists; the user decides.**
3. **No unnecessary data collection.**
4. **No automatic consequential submissions.**
5. **No CAPTCHA bypass.**
6. **Never pretend the AI is always right.**
7. **Always show where an answer came from.**
8. **Let users correct the system — corrections compound.**

### 11.1 The hard "no" list

Print this. These rules are the product's integrity, not a wishlist.

| Rule | Reason |
|---|---|
| ❌ **No batch / bulk submission** | One form, one click, human-initiated. Ever. |
| ❌ **No CAPTCHA solving services** | No 2Captcha. No residential proxies. No Turnstile bypass. The human solves; we pause and wait. |
| ❌ **No arbitrary third-party sites** | Allowlist: forms the user owns + forms they are authorised for. |
| ❌ **No auto-submit without review** | The human gate is the product. |
| ❌ **No stored passwords or OTPs** | The user types their own OTP. We never see or store it. |
| ❌ **No government KYC / exam / financial attestation forms** | Blocklist by design. |
| ✅ **Hard rate cap** | ~30 submissions/day/user. A spam tool needs thousands. This cap makes us structurally incapable of being one. |

**Why this is strategic, not defensive:** these rules are what allow us to hold sensitive
student data — marksheets, resume, photo, signature, category, income — with a straight face.
They are the reason we can publish the code while Browse AI, Axiom, Strawberry, and 1Password
cannot. **This is the go-to-market.**

### 11.2 Privacy architecture

- **Local by default.** Form content is never transmitted.
- **The AI sees labels, not documents.** Mapping sends field labels + a profile summary.
- **The passphrase never leaves the device.** No recovery server, no backdoor.
- **Export and delete are first-class features**, not a support ticket.
- **Opt-in telemetry only**, and never containing form content.

### 11.3 India — DPDP Act 2023

- PII local by default; explicit opt-in for any sync
- Documented retention and deletion
- **A published privacy policy is a hard launch requirement**
- Consent must be affirmative and withdrawable

### 11.4 WhatsApp: 2026 compliance constraints

**Verify these before building Phase 4.** They materially change the architecture.

| Constraint | Detail |
|---|---|
| **AI Provider terms** | From **15 Jan 2026**, WhatsApp's ToS restrict "AI Providers" offering general-purpose AI assistants on the Business Platform **only where Meta is legally required to permit it.** Availability is jurisdiction-dependent, not just product-dependent. |
| **AI Provider billing** | From **16 Feb 2026**, Meta charges AI Providers for **every non-template message.** Launched EU/UK/Italy; withdrawn in most on 13 May 2026; **Brazil (+55) currently live.** Webhook pricing category `general_purpose_ai`; analytics category `AI_BOT`. |
| **India INR migration** | India WABAs must migrate to **INR billing by 31 Dec 2026.** From **1 Jan 2027**, Meta will **not deliver messages** from non-INR WABAs. |
| **Free entry point** | Click-to-WhatsApp ad replies from the **Android/iOS app** (not desktop/web) open a **72-hour free window.** Cheapest acquisition channel available. |
| **24-hour window** | Free-form replies only inside the customer service window. Outside it, approved templates only. |

**Design consequence:** the conversational intake engine must be **channel-swappable**. WhatsApp,
web, Telegram, and future IVR all sit on the same intake core.

> Never let one channel's policy become existential for the product.

This is also why WhatsApp is a *layer*, not the foundation — even though it may become the
most *used* surface.

### 11.5 Open-source posture

Being open source is the **trust mechanism**, not a growth hack.

- Anyone can verify that form content is never transmitted
- Anyone can verify there is no CAPTCHA bypass
- Anyone can audit what leaves the device

**Publish the client-side storage path first, and the extension before the backend exists.**
The auditable surface is the product.

**Licensing** — pick one before you have users. Retrofitting a license change after adoption
is a community incident.

| Asset | License | Why |
|---|---|---|
| Client (extension, web app, vault, mapping) | MIT or AGPL | Auditable is the point |
| Mapping model weights / training data | Open, documented | Keeps the moat public where it should be |
| Any hosted service | Proprietary | Only if one ever exists |

### 11.6 Security checklist before public beta

- [ ] Vault encryption verified, passphrase never transmitted
- [ ] No content-script logging of field values
- [ ] Telemetry verified empty of form content (publish the audit)
- [ ] Privacy policy published
- [ ] Data export + hard delete working end-to-end
- [ ] Dependency audit clean (`npm audit`, lockfile pinned)
- [ ] No CAPTCHA/proxy libraries anywhere in the tree — verified by grep
- [ ] Rate limiter active and tested
- [ ] Content script CSP compliant, no `eval`

---

## 12. Competitive Landscape & Moat

### Landscape

| | Strawberry | Browse AI | Axiom.ai | Refrain |
|---|---|---|---|---|
| Audience | Enterprise | Businesses | Businesses | Anyone, students first |
| Price | $20–250/mo | $19–500/mo | $15–250/mo | Free → paid |
| Shape | Full browser | Build-a-robot | Build-a-bot | **Zero-config companion** |
| Inference | Metered credits | Cloud | Cloud | **Local-first, ~₹0** |
| Data | Cloud, SOC 2 | Cloud | Cloud | **Local, encrypted, auditable** |
| Document extraction | No | No | No | **Yes** |
| Remembers your applications | No | No | No | **Yes — the Setlist** |
| Open source | No | No | No | **Yes** |

### Why they can't follow us into students

Credit-metered competitors pay inference cost per action. Filling 60 forms a month is exactly
the high-frequency usage that destroys their unit economics. They *have* to keep users on a
slow, metered, expensive path.

Our local-first model means ~₹0 marginal cost per fill. **We can serve the highest-frequency
user in the market; they structurally cannot.**

> That is a moat made of arithmetic, not features.

### What they have that we don't

Be honest. Strawberry has: a real team, native macOS + Windows apps, polished onboarding, an
enterprise sales motion, and SOC 2. **Do not compete on any of those.**

### The threat we cannot ignore

**Chrome and Edge are shipping native AI form filling.** Free, pre-installed, improving fast.
Their generic version will always be worse at Indian portal specifics.

**Counter-arguments:**

- Native features are **not document-derived**
- Not **portal-aware**, not **tracker-aware**
- **Not auditable** — cloud-bound, closed
- **Generic by design** — Google will never ship "Maharashtra HSC marksheet → NSP portal"

If a student can satisfy 80% of forms with Chrome's built-in feature in one click, Refrain must
be obviously better on the other 20% *and* own what native features never will: **document
extraction, the Setlist, and Encore re-fills.**

### Moat

| Moat | Why it compounds |
|---|---|
| **Mapping dataset** | Every resolved field-label → value pair improves routing. Students annotating messy Indian form labels build a dataset that **cannot be scraped.** This is the real asset. |
| **Encore effect** | The longer a user's history, the more valuable one-tap re-fills become. Switching cost grows with use. |
| **Institutional lock-in** | Placement-cell dashboards + student histories = annual contract. *(Secondary; not required for v1.)* |
| **OSS trust** | Auditable local-first privacy is *verifiable*. Competitors can only *claim* it. |
| **Local-first COGS** | ~₹0 marginal inference cost → we serve high-frequency users credit-metered competitors cannot. |

### Building the mapping dataset deliberately

Do not leave this to chance. From Phase 2:

1. Log every field label the extension sees (locally, opt-in share)
2. Every user correction becomes a labelled training pair
3. Anonymise and aggregate into a public dataset
4. Ship the dataset with the repo

**This is the one asset that gets better while competitors watch.**

---

## 13. Business Model

Not the first milestone — but designed for from day one.

| Tier | Price | Purpose |
|---|---|---|
| **Free** | ₹0 | Core filling + Setlist. The distribution engine. |
| **Pro** | ₹99–149/mo | Unlimited forms, document parsing, bulk Variations, priority support |
| **Institutional** | ₹15k–40k/college/yr | Placement-cell dashboard, batch management, admin, SSO, support |
| **Agency** | ₹1.5k–3k/seat/mo | Multi-client profile management |

**Honest read:** B2C student revenue alone does not work. Free users are marketing spend we
chose. **Institutional licences are the business** — but that is Phase 6+, and only if the
consumer product proves retention first.

### Distribution

| Channel | Notes |
|---|---|
| **Chrome Web Store** | Primary. Install friction is a real risk — see §18. |
| **College placement cells** | One email ≈ 300 students. Also the future paying customer. |
| **YouTube** | *"How I apply to 50 scholarships in 20 minutes"* — students share this themselves |
| **Open source + audits** | Security-conscious students won't install a random extension otherwise |
| **College WhatsApp groups, r/india, GitHub** | Peer-to-peer |

**The organic loop we want:** `"Bro, you need this. It fills forms for you."`

---

## 14. Roadmap

### Phase 0 — Foundation
Profile creation · document upload · saved answers · vault encryption

*Exit: a user can store everything about themselves.*

### Phase 1 — MVP ← **the only phase that matters right now**
Google Forms + basic web forms. Detect fields · understand fields · map to profile · fill ·
review · **user submits**

*Exit: one form filled end-to-end. "I found 14 fields."*

### Phase 2 — Genuinely good
Semantic matching · document understanding · file uploads · reusable answers · multi-page
forms · conditional fields · confidence indicators · missing-info detection · form history

*Exit: it works on the messy real portals students actually use.*

### Phase 3 — The Setlist
Tracker · status · deadlines · notes · history · reminders · **Encore re-fills**

*Exit: users return weekly.*

### Phase 4 — WhatsApp companion
Profile create/update · document upload · saved answers · application updates · reminders

*Exit: the product is reachable without opening the web app.*

⚠️ Requires the §11.4 compliance work first — AI Provider terms, AI Provider billing, and INR
WABA migration by **31 Dec 2026**.

### Phase 5 — More websites
Internship platforms · job applications · scholarship portals · event systems · form builders.
**Support what users actually use. Do not chase every website.**

### Phase 6 — Job seeker expansion
Resume versions · cover letters · application tracking · interview prep · follow-ups

**Do not let this become a generic job platform.** The core remains: *your information →
application → action.*

### Phase 7 — General personal form companion
Students → job seekers → professionals → freelancers → everyone.

### Deliberately not doing

- ❌ Native desktop app (two platforms, installer hell, browser tax)
- ❌ Credit-metered pricing (we have ~₹0 COGS — metering only punishes us)
- ❌ Comparison-page SEO strategy (mature-team play, no traffic to convert)
- ❌ Breadth — slide decks, transcripts, prospecting, routines. One thing, filled perfectly.
- ❌ Telegram before WhatsApp
- ❌ Institutional tier before consumer retention is proven

---

## 15. Success Metrics

### The first milestone is not revenue.

| Stage | Metric |
|---|---|
| **1** | **100 people who actually use it** (not just install it) |
| **2** | 1,000 active users |
| **3** | 10,000 |

### The only question that matters at every stage

> **Are people coming back?**

Because **retention is the proof the problem is real.**

### Secondary metrics

| Metric | Target | Why |
|---|---|---|
| Forms completed per active user per week | 2+ by week 8 | Proves repeated use, not novelty |
| **Encore rate** | rising | % of sessions that are a one-tap re-fill — the compounding signal |
| Correction rate on auto-filled fields | **decreasing** | Proves the mapping model is learning |
| Setlist return visits | rising | Users come back to check status, not to fill |
| Organic referral | qualitative | `"Bro, you need this."` |

**Correction rate decreasing is the most important of these.** It is the only objective proof
that the moat is building.

### The value test

> If a 20-field form takes someone 5 minutes manually, and Refrain reduces it to 30 seconds of
> review, that is a proven value proposition. **Nothing else matters.**

---

## 16. Long-Term Vision

> **Today: Forms.**
> **Tomorrow: Applications.**
> **Eventually: Personal digital workflows.**

### The asset underneath

Not creepy — conceptual. Refrain maintains a **personal information graph**:

```
                        USER
                         │
          ┌──────────────┼──────────────┐
          │              │              │
      Education     Experience      Personal
          │              │              │
      Documents     Projects      Contact
          │              │              │
          └──────────────┼──────────────┘
                         │
                   Applications
                         │
            ┌────────────┼────────────┐
            │            │            │
        Internship      Job      Scholarship
```

Refrain understands the **relationships** between these things. That is worth far more than
autofill.

### End state

> A personal AI companion that understands a user's information once and helps them reuse it
> across the digital forms and applications they encounter throughout their life.

---

## 17. Operating Principles

1. **User owns their data.**
2. **AI assists; the user decides.**
3. **No unnecessary data collection.**
4. **No automatic consequential submissions.**
5. **No CAPTCHA bypass.**
6. **Never pretend the AI is always right.**
7. **Always show where an answer came from.**
8. **Let users correct the system — corrections compound.**

---

## 18. Open Questions & Risks

### Open questions — priority order

| # | Question | Why it matters |
|---|---|---|
| 1 | Which single vertical first — **scholarships**, **internships**, or **event/feedback forms**?** | Scholarships = highest pain, most repetitive. Feedback forms = easiest first win. Everything in Phase 2 depends on this. |
| 2 | Is Chrome Web Store distribution enough? | Install friction and review latency may be a bigger risk than the build. Alternative: sideload for the placement-cell pilot, CWS for scale. |
| 3 | Is the mapping dataset a real moat or a narrative? | What would it take to reach a genuinely hard-to-replicate dataset? Plan the collection mechanism in Phase 2, not later. |
| 4 | How far does local inference actually go? | If Chrome's Prompt API handles 90% of mappings, we never touch a cloud bill. If 40%, the architecture and cost model change. **Test this in week 1.** |
| 5 | Is the Setlist enough retention, or do we need deadlines/reminders? | Reminders mean a notification surface: permissions, reliability, spam perception. Real cost. |
| 6 | When does the institutional tier become worth building? | Too early = wasted effort. Too late = missed revenue. Depends on #1 and #2. |
| 7 | Will Chrome's native AI autofill eat this? | If a student satisfies 80% of forms with the built-in feature, we must be obviously better on the other 20% *and* own document extraction, the Setlist, and Encore. |
| 8 | Which license, chosen before users? | MIT for adoption, AGPL for protecting the auditable surface. Retrofitting after adoption is a community incident. |
| 9 | Do we need a passphrase at all for v1? | Passphrase friction will hurt the 100-user target. Zero-knowledge encryption and zero onboarding are in tension. Resolve before building the vault. |
| 10 | What is our position on form *ownership*? | "Forms you own + forms you're authorised for" needs a concrete rule. Session-scoped access? Explicit grant per domain? Undefined, it becomes a loophole. |

### Risks

| Risk | Severity | Mitigation |
|---|---|---|
| Chrome Web Store rejects or delays the extension | Medium | Sideload for pilots; keep permissions minimal and justified |
| Native browser AI autofill commoditises the category | **High** | Document extraction + Setlist + Encore. Own the vertical they won't build. |
| Local inference proves too weak | Medium | BYOK cloud fallback; better local models; rule-based first |
| Students won't install an extension that touches sensitive data | Medium | Open source + published privacy policy + the "no" list as marketing |
| A portal changes its HTML and we break silently | Medium | Isolate content script; ship failure logging; add adapters only on demand |
| We drift into building five products | **High** | The §4 list exists for this. Re-read it monthly. |
| Burnout on a solo build | High | Phase 1 is genuinely ~2 weeks. Stop before it becomes 6 months. |

---

## 19. Decisions Made

*Recorded here so they don't get relitigated.*

| Date | Decision | Rationale |
|---|---|---|
| 2026-10-05 | **Renamed Reprise → Refrain** | *Refrain* = the part of a song that keeps coming back; also *refrain from* = to hold back, which is the §11 human gate. Double meaning encodes the hardest constraint. Verified clear of funded software (16 candidates checked; Glean, Weave, Fold, Winnow, Refold, Riff, Discern, Glint, Spire, Weld, Pluck, Heron, Talon, Perch all collided) |
| 2026-10-05 | **Word system is musical, not mechanical** | Verse / Variation / Motif / Setlist / Encore / Tempo / Fermata / Attacca. Nine free nouns that scale. Replaces the old Reel / Theme / Restatement / Cadence |
| 2026-10-05 | **Mascot states are named** | Rest · Listening · **Fermata** · Attacca. Fermata names the human gate, so the §11 principle appears in the UI, not just the policy |
| 2026-10-05 | **Rejected "OneFill"** | It names the *mechanism*, not the value — which frames us as the commodity autofill we are disrupting. `Fill` is owned by Apple AutoFill, Chrome AutoFill and 1Password's Fill. Already rejected once as *FillOnce* in the original §1; not relitigating |
| 2026-10-04 | **Extension + side panel**, not a browser | Same-origin policy makes a web-only app impossible; a browser is 18 months we don't have |
| 2026-10-04 | **Students first, not students only** | Students are the beachhead, not the positioning ceiling |
| 2026-10-04 | **Local-first, no credits** | ~₹0 COGS is what makes a ₹0 price point survivable, and it is a structural moat |
| 2026-10-04 | **Open source** | Verifiable privacy is the trust mechanism and the distribution strategy |
| 2026-10-04 | **WhatsApp is a layer, not the foundation** | AI Provider ToS + per-message billing + Jan 2027 INR deadline make it too risky to build on |
| 2026-10-04 | **Hard "no" list adopted** | Structural integrity + marketing story + the reason we can be open source |

---

## 20. Build Order

*The actual to-do list for starting. Do these in order. Do not skip ahead.*

### Phase 1, in build order

1. **Monorepo scaffold** — pnpm + Turborepo + TypeScript + Tailwind + design tokens
2. **`packages/ui`** — mascot component with its four states
3. **Content script** — extract form schema from Google Forms
4. **Side panel** — render *"I found 14 fields"*  ← **first demo checkpoint**
5. **`packages/vault`** — encrypted profile store
6. **Mapping** — label → profile field, **rule-based first**
7. **Review screen** — read-only render of proposed values with provenance
8. **Write values to the page** — native setter + events (§6.5 Gotcha 1)
9. **User presses submit** ← **MVP reached**
10. **Log to the Setlist**

### The two rules that matter

**Ship order, every phase:**
```
Form Reader → Smart Mapping → Review Screen → Submit
```

Get one full form end-to-end before anything else. **A tool that fills 14 of 16 fields and asks
nicely beats one that promises everything and breaks on a JSP portal.**

**Rule-based mapping before LLM mapping.** Get the plumbing right with
`if label contains "email"` first. Add intelligence only once the end-to-end path is proven.

### Week 1 deliverables (realistic)

- [ ] Monorepo runs, side panel opens in Chrome
- [ ] Mascot renders all four states
- [ ] Google Forms reader reports field count and labels
- [ ] One form filled end-to-end with a human pressing submit

---

## 21. Homepage Copy

### Hero

> **Stop filling the same information again and again.**
>
> ### Answer once. Fill everywhere.
>
> Your personal AI companion for forms, applications and repetitive online workflows.

**Buttons:** `Try it free` · `View on GitHub`

### Then immediately show a real demo

```
FORM

Full Name          ✓  Rohit Jadhav
Email              ✓  ******@gmail.com
College            ✓  Atharva University
Branch             ✓  Electronics & Computer Science
Resume             ✓  Rohit_Resume.pdf
10th Percentage    ✓  87.4%   (from marksheet.pdf)

        [ Review & Fill ]
```

**This communicates the product in seconds.** A feature list does not.

### Explanation section

> Save your information, documents and answers once. When you encounter a repetitive form,
> your companion understands the questions, finds the right information, fills the form,
> and lets you review everything before you submit.

---

## 22. Shareable Brief

*Forwardable. No internal notes, no doubts. Designed to invite specific feedback.*

---

### Refrain

**Your personal AI form companion. Answer once. Fill everywhere.**
Pre-build · open source · `refrain.layerflow.dev`

**The name.** A *refrain* is the part of a song that keeps coming back — and *refrain from*
means to hold back. Your information returns every time it is asked; the product never moves
without you. The sub-products are all music: *Verse* (a saved answer), *Variation* (one
adapted to a specific question), *Setlist* (your application tracker), *Encore* (a one-tap
re-fill), *Fermata* (the pause where you decide).

**The problem.** An average student applies to 50–150 forms a month. Each asks the same 20
questions in different words (`Full Name` / `Candidate Name` / `Name of Student`), re-asks
known facts, wants the same documents re-uploaded, spans 4–6 page wizards, and ends in a
CAPTCHA. Password managers match exact labels and miss. Browser automation is priced per
runtime-hour and requires building an automation. **Nobody has solved document-derived data** —
reading "87.4%" out of a marksheet PDF into the right box.

**The product.** A browser companion that reads the form, maps it to your profile using
meaning not string matching, shows a review screen with the provenance of every value, fills
it when you approve, pauses at the CAPTCHA for you to solve, and logs everything to your
private Application Tracker.

**Audience.** Primary: Indian college students (18–23). Secondary: **college placement cells**
(300–1,000 applications per cycle, currently solved with spreadsheets) — *this is who pays*.
Then job seekers, then professionals, then everyone. Students are the beachhead, not the
ceiling.

**Business model.** Free ₹0 (distribution engine) · Pro ₹99–149/mo · Institutional
₹15k–40k/college/yr · Agency ₹1.5k–3k/seat/mo. B2C student revenue alone doesn't work; free
users are marketing spend we chose, institutional licences are the business.

**Tech stack.** pnpm + Turborepo + TypeScript · React + Vite + Tailwind + shadcn/ui + Zustand
+ React Hook Form/Zod + XState · Chrome MV3 + Plasmo with a **vanilla-TS** content script and
`all_frames: true` · Dexie (IndexedDB) + AES-GCM WebCrypto · Chrome Prompt API (Gemini Nano)
→ Ollama → BYOK cloud · pdf.js + Tesseract · Cloudflare Workers backend that **stores no form
content**.

**UI.** Warm, rounded, friendly — Duolingo, not Salesforce. Cartoon mascot as a **state
machine, not a personality**: Idle / Reading / Needs you / Done. Motion while working, silence
while the user types. It should make someone smile *once*, then become invisible. Mascot UI is
culturally native to Indian consumer apps and is what students screenshot and share — our
screenshot is our marketing. The **Review Screen is the hero screen**.

**Position.**

| | Strawberry | Browse AI | Axiom.ai | Refrain |
|---|---|---|---|---|
| Audience | Enterprise | Businesses | Businesses | Anyone, students first |
| Price | $20–250/mo | $19–500/mo | $15–250/mo | Free → paid |
| Shape | Full browser | Build-a-robot | Build-a-bot | **Zero-config companion** |
| Inference | Metered credits | Cloud | Cloud | **Local-first, ~₹0** |
| Data | Cloud, SOC 2 | Cloud | Cloud | **Local, encrypted, auditable** |
| Document extraction | No | No | No | **Yes** |
| Remembers applications | No | No | No | **Yes** |
| Open source | No | No | No | **Yes** |

**Structural advantage:** credit-metered competitors pay inference cost per action. Filling 60
forms a month destroys their unit economics — they must keep users slow and metered. Our
local-first model is ~₹0 marginal. **We can serve the highest-frequency user in the market;
they structurally cannot.**

**Threat:** Chrome and Edge are shipping native AI form filling, free and pre-installed. But
generic — not document-derived, not portal-aware, not auditable. Google will never ship
"Maharashtra HSC marksheet → NSP portal."

**Moat.** Mapping dataset (student-annotated field labels, unscrapeable) · Encore one-tap
re-fills · OSS verifiable trust · local-first COGS · institutional lock-in later.

**Out of scope.** ❌ browser ❌ autonomous agent ❌ RPA platform ❌ college ERP ❌ CRM ❌ job
board ❌ CAPTCHA bypass ❌ autonomous submission.
**Non-negotiable:** one form at a time, human-initiated · human solves CAPTCHAs, we wait · no
arbitrary third-party sites · no stored passwords or OTPs · no gov KYC or competitive-exam
forms · ~30 submissions/day cap. These aren't limitations — they're what let us hold
sensitive student data responsibly and be open source when incumbents can't.

**Why now.** Chrome side panel API (114+) shipped a persistent companion with page access as a
stable web API — this UI wasn't buildable before ~2023 without forking a browser. On-device
LLMs make local inference real: privacy-preserving *and* near-zero-cost. And every incumbent
is closed and cloud-based, so an auditable open-source client is a genuine differentiator.

**Roadmap.** 0 Foundation → **1 MVP (Google Forms end-to-end)** → 2 semantic mapping +
documents → 3 the Setlist → 4 WhatsApp layer → 5 more portals → 6 job seekers → 7 everyone.
Ship order always: `Form Reader → Smart Mapping → Review Screen → Submit`.

**Success.** Not revenue. **100 people who actually use it** → 1,000 → 10,000. The only
question each stage: *are people coming back?* Key metric: **correction rate on auto-filled
fields must decrease** — objective proof the moat is building.

**Five questions I'd like feedback on:**

1. **Is the institutional (placement cell) motion real, or am I inventing a buyer?** In India,
   who actually signs off — department, admin, or a student fee fund? Sales cycle length?
2. **Is Chrome Web Store distribution enough?** Install friction and review latency worry me
   more than the build does. Has anyone shipped a student consumer extension successfully this
   way?
3. **Is "local-first, zero credits" genuinely a moat, or just a pricing artifact?** Once Google
   ships native AI autofill, what stops them from doing this exact thing?
4. **How much does the mapping dataset actually matter?** Real compounding asset or nice
   narrative? What would it take to reach a genuinely hard-to-replicate dataset?
5. **Wedge check:** scholarships, internships, or event/feedback forms first? I lean
   scholarships (highest pain, most repetitive), but feedback forms are the easiest first win.

---

## 23. One-Line Summary

> **Refrain** is a personal AI companion that remembers your information once and helps you
> use it everywhere you repeatedly fill forms and applications — starting with students,
> expanding to job seekers, and eventually serving anyone.

---

*This document is the single source of truth. Update it, not your memory.*