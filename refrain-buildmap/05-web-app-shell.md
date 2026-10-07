# Chapter 5 — The Web App Shell

> **Day 5 · Goal: every screen exists, navigates, and is empty. No product logic yet.**
>
> Front end first. You build the whole skeleton — routing, layout, every page — so that from
> Chapter 6 onward you are *filling in* a working app rather than *constructing* one. A shell
> that navigates is the cheapest possible feedback loop, and it costs one day.

---

## Understand this first

### Why the shell comes before the vault

It is tempting to build in dependency order: vault → mapping → UI. That produces a UI you
design *around* the implementation, which is how you end up with a settings page that is really
just a JSON dump.

**Build the shell first and the vault designs itself around a question you have already
answered: where does the user edit a fact?** If `/profile` exists as a route before you write
a line of vault code, you will build a fact editor. If it does not, you will build a
`localStorage` dump.

Every chapter from here is "fill in a route that already exists." That is the whole point of
this chapter.

### Two surfaces, two jobs — do not let them blur

| Surface | Job | Mental model | Input |
|---|---|---|---|
| **Web app** | **Edit** your data | A settings app | Keyboard, mouse, hours |
| **Side panel** | **Act** on a form | A companion | Pointer, seconds, mid-form |

A student opens the web app on a Sunday evening to fix their address. They open the side panel
on a Tuesday to fill a form. **The web app is for setup; the panel is for the moment.**

Design consequence: the web app can afford a table, a sidebar, and a settings gear. The panel
cannot — it is 320px wide and the user is looking at a form in the other half of the screen.

> §6.2 says "proper web *is* compatible." That compatibility is cheap *because* of this split:
> the side panel imports the same `ReviewRow`, the same `ProvenanceChip`, and the same Zustand
> store as the web app. Two doors, one building.

### Routing is a contract, not navigation

Every route here is a **place the user can be deep-linked to, bookmarked, and reached by a
support message.** When someone emails "I can't see my documents," you send them a link.

That means: no route without a real path, no state that only exists inside a component, and a
404 that says where the user can go next. You will thank yourself in Chapter 8 when you are
debugging a Setlist entry from a URL.

---

## Step 1 — Router

```bash
pnpm --filter @refrain/web add react-router
```

`react-router` v7 is the current version. **Note the API change:** v7 removed the `react-router-dom`
split — you now import from `react-router` directly.

```tsx
// apps/web/src/main.tsx
import React from "react"
import ReactDOM from "react-dom/client"
import { createBrowserRouter, RouterProvider } from "react-router"
import "@refrain/ui/tokens.css"
import "./styles/app.css"

import { AppShell } from "./routes/AppShell"
import { UnlockGate } from "./routes/UnlockGate"
import { Dashboard } from "./routes/Dashboard"
import { Profile } from "./routes/Profile"
import { Documents } from "./routes/Documents"
import { Setlist } from "./routes/Setlist"
import { Verses } from "./routes/Verses"
import { Settings } from "./routes/Settings"
import { NotFound } from "./routes/NotFound"
import { RouteError } from "./routes/RouteError"

const router = createBrowserRouter([
  {
    // The gate sits OUTSIDE the shell. A locked user has no nav,
    // because there is nothing they can do yet.
    element: <UnlockGate />,
    children: [
      {
        path: "/unlock",
        element: <Unlock />,
      },
    ],
  },
  {
    element: <AppShell />,
    children: [
      { index: true,           element: <Dashboard /> },
      { path: "profile",       element: <Profile /> },
      { path: "documents",     element: <Documents /> },
      { path: "documents/:id", element: <DocumentDetail /> },
      { path: "setlist",       element: <Setlist /> },
      { path: "verses",        element: <Verses /> },
      { path: "settings",      element: <Settings /> },
      { path: "*",             element: <NotFound /> },
    ],
  },
])

ReactDOM.createRoot(document.getElementById("root")!).render(
  <React.StrictMode>
    <RouterProvider router={router} />
  </React.StrictMode>,
)
```

**Three structural decisions:**

**The gate is a separate route branch, not a redirect.** `<UnlockGate />` returns `<Outlet />`
if unlocked, or renders the unlock form if not. Putting it outside `AppShell` means a locked user
cannot see the nav — which is correct, because they cannot do anything.

**`:id` is a route param, not a component `useState`.** `/documents/abc123` is deep-linkable
and works on a fresh reload. A document detail view that only exists as in-component state is
a bug you will find during support.

**`*` is a real 404, not a redirect to `/`.** Silently redirecting hides broken links from you
*and* from your users. The 404 screen lists the real routes.

---

## Step 2 — The unlock gate

```tsx
// apps/web/src/routes/UnlockGate.tsx
import { useEffect, type ReactNode } from "react"
import { Outlet } from "react-router"
import { vault } from "@refrain/vault"

export function UnlockGate() {
  // The vault state lives outside React. Subscribe to it.
  useEffect(() => vault.subscribe(() => forceUpdate()), [])

  if (!vault.isReady) return <Onboarding />
  if (!vault.isUnlocked) return <Unlock />

  return <Outlet />
}
```

`vault.subscribe()` rather than reading `vault.isUnlocked` during render — the vault is a plain
class outside React, so you need a bridge. `useSyncExternalStore` is the correct primitive and
Chapter 6 formalises it. **Do not poll.** Polling `isUnlocked` every 500ms is how you end up
with a vault that unlocks visually a second late on every page.

---

## Step 3 — The shell

```tsx
// apps/web/src/routes/AppShell.tsx
import { NavLink, Outlet } from "react-router"
import { Mascot } from "@refrain/ui"

const NAV = [
  { to: "/",          label: "Overview",  glyph: "◈" },
  { to: "/profile",   label: "Profile",   glyph: "✓" },
  { to: "/documents", label: "Documents", glyph: "📄" },
  { to: "/setlist",   label: "Setlist",   glyph: "♪" },
  { to: "/verses",    label: "Verses",    glyph: "✎" },
] as const

export function AppShell() {
  return (
    <div className="min-h-dvh grid grid-cols-[248px_1fr] bg-surface">
      {/* ── Sidebar ── */}
      <aside className="flex flex-col gap-1 border-r border-surface-muted p-4">
        <div className="mb-6 flex items-center gap-3 px-2">
          <Mascot state="idle" className="!p-0 !bg-transparent" />
          <span className="text-lg font-semibold tracking-tight text-ink">Refrain</span>
        </div>

        <nav aria-label="Main" className="flex flex-col gap-0.5">
          {NAV.map((item) => (
            <NavLink key={item.to} to={item.to} end={item.to === "/"}
              className={({ isActive }) =>
                `flex items-center gap-3 rounded-card px-3 py-2 text-sm transition-colors ${
                  isActive
                    ? "bg-brand-500/10 font-medium text-brand-700"
                    : "text-ink-muted hover:bg-surface-muted"
                }`}>
              <span aria-hidden>{item.glyph}</span>
              {item.label}
            </NavLink>
          ))}
        </nav>

        {/* Vault state is always visible. Never make people wonder
            whether their data is currently readable. */}
        <div className="mt-auto rounded-card bg-surface-muted p-3 text-xs text-ink-muted">
          <p className="font-medium text-ink">Vault unlocked</p>
          <p className="mt-1">Everything on this device. Nothing sent.</p>
          <button className="mt-2 font-medium text-brand-700">Lock now</button>
        </div>
      </aside>

      {/* ── Main ── */}
      <main className="min-w-0 overflow-y-auto">
        <div className="mx-auto max-w-3xl p-8">
          <Outlet />
        </div>
      </main>
    </div>
  )
}
```

**Five decisions:**

**The lock state is permanently visible in the sidebar.** A user who does not know their vault is
encrypted will not trust it. This costs 40px and buys the entire §11 argument.

**`end` on the `/` link.** Without it, `/` matches every route and Overview stays highlighted
forever. Small thing, immediately noticeable.

**`248px` fixed sidebar.** Content needs a defined left edge. A fluid sidebar that reflows at
every breakpoint makes every screen's layout a negotiation.

**`min-w-0` on `<main>`.** Without it, a wide table inside `<main>` forces horizontal scroll on
the *whole grid*, and the sidebar slides off screen. This is the single most common Tailwind
grid bug.

**`<nav aria-label="Main">`.** Two navs in one app is common. The label makes the landmark
distinguishable to a screen reader.

---

## Step 4 — Every screen, stubbed

Each route exists and renders. **No product logic yet.** This is the point.

```tsx
// apps/web/src/routes/Dashboard.tsx
import { Mascot } from "@refrain/ui"
import { StatCard } from "../components/StatCard"

export function Dashboard() {
  return (
    <div>
      <Mascot state="idle" />

      <h1 className="mt-6 text-2xl font-semibold text-ink">Good to see you</h1>
      <p className="mt-1 text-sm text-ink-muted">
        Refrain will fill forms from what is already here.
      </p>

      <div className="mt-8 grid grid-cols-3 gap-3">
        <StatCard label="Profile fields" value="—" hint="of 24 known" to="/profile" />
        <StatCard label="Documents"     value="—" hint="none uploaded"  to="/documents" />
        <StatCard label="Setlist"       value="—" hint="forms prepared" to="/setlist" />
      </div>

      {/* §7 Pillar 4 explained to a first-time user, in the place
          they will actually read it. */}
      <section className="mt-8 rounded-panel bg-surface-muted p-5">
        <h2 className="font-medium text-ink">How Refrain works</h2>
        <ol className="mt-3 flex flex-col gap-2 text-sm text-ink-muted">
          <li>1. Add your details once, or drop in a marksheet.</li>
          <li>2. Open any form. The side panel appears automatically.</li>
          <li>3. Refrain fills what it knows and shows you where each value came from.</li>
          <li>4. You check it, then you press submit. Refrain never does.</li>
        </ol>
      </section>
    </div>
  )
}
```

```tsx
// apps/web/src/routes/Profile.tsx
export function Profile() {
  // Chapter 6 replaces this with the real fact graph.
  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Your profile</h1>
      <p className="mt-1 text-sm text-ink-muted">
        One fact, answered once, used everywhere.
      </p>

      <div className="mt-8 flex flex-col gap-4">
        {["Name", "Email", "Phone", "Address", "CGPA", "10th %", "12th %", "University"]
          .map((label) => (
            <div key={label} className="flex items-center gap-4 rounded-card border border-surface-muted p-4">
              <span className="w-32 text-sm text-ink-muted">{label}</span>
              <span className="flex-1 text-sm text-ink-muted">— empty —</span>
              <span className="chip chip--dashed">you type this</span>
            </div>
          ))}
      </div>
    </div>
  )
}
```

```tsx
// apps/web/src/routes/Documents.tsx
export function Documents() {
  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Documents</h1>
      <p className="mt-1 text-sm text-ink-muted">
        Drop in a marksheet and Refrain will read it — once — and ask you to check each field.
      </p>

      {/* The drop zone is the whole page. Not a button in a corner. */}
      <div className="mt-8 grid min-h-64 place-items-center rounded-panel border-2 border-dashed border-surface-muted p-8 text-center">
        <div>
          <p className="font-medium text-ink">Drop a PDF here</p>
          <p className="mt-1 text-sm text-ink-muted">
            Stays on this device. Encrypted before it is stored.
          </p>
        </div>
      </div>
    </div>
  )
}
```

```tsx
// apps/web/src/routes/Setlist.tsx
export function Setlist() {
  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Your setlist</h1>
      <p className="mt-1 text-sm text-ink-muted">Every form you have prepared.</p>
      <p className="mt-8 rounded-card bg-surface-muted p-4 text-sm text-ink-muted">
        Nothing here yet. Refrain adds a form when you scan one from the side panel.
      </p>
    </div>
  )
}
```

```tsx
// apps/web/src/routes/Verses.tsx
export function Verses() {
  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Verses</h1>
      <p className="mt-1 text-sm text-ink-muted">
        Answers you reuse often. A verse adapts to what the form is asking for.
      </p>
      {/* §7 Pillar 3. This is the feature most competitors lack
          entirely: a saved "About me" that shortens for a
          200-character box and expands for a 2000-character one. */}
      <div className="mt-8 flex flex-col gap-3">
        {["Why I want this scholarship", "Why this role", "Tell us about yourself"]
          .map((t) => (
            <div key={t} className="rounded-card border border-surface-muted p-4">
              <p className="font-medium text-ink">{t}</p>
              <p className="mt-1 text-sm text-ink-muted">— not written yet —</p>
            </div>
          ))}
      </div>
    </div>
  )
}
```

```tsx
// apps/web/src/routes/Settings.tsx
export function Settings() {
  return (
    <div>
      <h1 className="text-2xl font-semibold text-ink">Settings</h1>
      <div className="mt-8 flex flex-col divide-y divide-surface-muted">
        {[
          ["Change passphrase", "You will need the old one"],
          ["Sync across devices", "Off. Optional, and end-to-end encrypted"],
          ["Inference engine", "On-device · Chrome"],
          ["Export everything", "A decrypted archive, on your machine"],
          ["Delete vault", "Permanently removes every fact and document"],
        ].map(([label, hint]) => (
          <div key={label} className="flex items-center justify-between py-4">
            <div>
              <p className="text-sm font-medium text-ink">{label}</p>
              <p className="text-xs text-ink-muted">{hint}</p>
            </div>
            <button className="rounded-card border px-3 py-1.5 text-xs font-medium text-ink">
              Manage
            </button>
          </div>
        ))}
      </div>
    </div>
  )
}
```

> **The settings list is written before the settings exist, and it is the design.** Once you
> see these six rows you know exactly what Phase 1 owes you. Two of them — export and delete —
> are **not optional** and are easy to forget until you need them. Export is what a sceptical
> user looks for first, and its absence reads as "your data is held hostage."
>
> **Export and delete are trust features.** Build them early, not at the end.

---

## Step 5 — Error boundary and 404

```tsx
// apps/web/src/routes/RouteError.tsx
import { useRouteError } from "react-router"

export function RouteError() {
  const error = useRouteError() as Error | undefined

  return (
    <div className="grid min-h-dvh place-items-center bg-surface p-8">
      <div className="max-w-md rounded-panel bg-surface-muted p-6">
        <h1 className="font-semibold text-ink">Something went wrong here</h1>
        <p className="mt-2 text-sm text-ink-muted">
          {error?.message ?? "An unexpected error occurred."}
        </p>
        <button onClick={() => window.location.reload()}
          className="mt-4 rounded-card bg-brand-500 px-4 py-2 text-sm font-medium text-white">
          Reload
        </button>
      </div>
    </div>
  )
}
```

> **Never show a stack trace to a user.** Never show raw `error.message` either — it can
> contain internal paths. Log it to the console for yourself; show the user a sentence they can
> act on. This is a product holding marksheets; a leaked path is a small but real disclosure.

```tsx
// apps/web/src/routes/NotFound.tsx
import { Link } from "react-router"

export function NotFound() {
  return (
    <div className="py-20 text-center">
      <h1 className="text-2xl font-semibold text-ink">No page here</h1>
      <p className="mt-2 text-sm text-ink-muted">
        That link does not point anywhere in Refrain.
      </p>
      <div className="mt-6 flex justify-center gap-3">
        <Link to="/" className="rounded-card bg-brand-500 px-4 py-2 text-sm font-medium text-white">
          Overview
        </Link>
        <Link to="/settings" className="rounded-card border px-4 py-2 text-sm font-medium text-ink">
          Settings
        </Link>
      </div>
    </div>
  )
}
```

---

## Step 6 — Responsive

You will open this on a phone. Probably more often than you expect — a student editing their
address between classes.

```tsx
// Sidebar collapses to a bottom tab bar under 768px.
<div className="min-h-dvh grid grid-cols-1 md:grid-cols-[248px_1fr]">
  <aside className="hidden md:flex ...">{/* sidebar */}</aside>
  <nav className="md:hidden sticky bottom-0 flex border-t bg-surface ...">
    {/* 5 icon tabs, safe-area padded */}
  </nav>
</div>
```

```css
/* Respect the notch. Without this the bottom tabs sit under the
   home indicator on an iPhone and are literally untappable. */
nav.md\:hidden {
  padding-bottom: env(safe-area-inset-bottom, 0px);
}
```

> **`env(safe-area-inset-bottom)` is not optional on iOS.** A bottom tab bar without it is
> underneath the home indicator. Test on a real device, not just DevTools' device emulation —
> the emulation does not draw the indicator by default.

---

## Step 7 — Commit

```bash
git add -A
git commit -m "feat(web): app shell, routing, 7 screens stubbed, error boundary, 404"
```

Add one row to `refrain.md` §19:

| Date | Decision | Why |
|---|---|---|
| 2026-10-XX | Shell before vault | Building the screens first forced a real fact editor instead of a settings dump. Every later chapter fills in a route that already exists. |

---

## Your 60/40 split for this chapter

| Task | Who |
|---|---|
| **The route tree and which routes exist** | **You.** This is the information architecture of the product |
| **The Settings list** (six rows, before they exist) | **You.** It is the design |
| `AppShell` markup and the sidebar | **You**, then let OpenCode tidy the classNames |
| Screen copy — the headings and the "How Refrain works" steps | **You.** Copy is design |
| `RouteError` / `NotFound` markup | **OpenCode** |
| Responsive breakpoints and the bottom tab bar | **OpenCode** |
| `StatCard` component | **OpenCode** |
| `useSyncExternalStore` bridge for the vault | **OpenCode** — see Chapter 6 for the pattern you want |

---

## Gotchas in this chapter

**`useRouteError` is undefined.** You are on React Router v6 API. v7 imports from
`react-router`, v6 from `react-router-dom`. Check which you installed.

**Every route shows the same page.** You forgot `element: <Profile />` on the `path: "profile"`
entry — the path is declared but nothing renders.

**The sidebar scrolls off horizontally when a table is wide.** You need `min-w-0` on `<main>`.
This is the #1 Tailwind grid overflow bug.

**The bottom tabs are untappable on an iPhone.** Missing `env(safe-area-inset-bottom)`.

**NavLink stays highlighted on every page.** You need `end` on the `/` route.

**A deep link to `/profile` 404s on a static server.** `createBrowserRouter` needs a history
fallback — every route must serve `index.html`. This bites on Netlify, Vercel, and Firebase
Hosting. Add the rewrite rule *now*, not at deploy time.

**Two `<nav>` elements.** Give each an `aria-label`. This surfaces immediately in an
accessibility audit and costs nothing.

**`Mascot` in the sidebar is 48px and dominates.** You passed `className="!p-0 !bg-transparent"`
but the SVG is still 48px. Wrap it in a sized div.

---

## Verify before moving on

- [ ] All 7 routes render and are navigable
- [ ] Deep-linking to each route works on a hard refresh
- [ ] `/nowhere` shows the 404 with working links
- [ ] A thrown error in a route shows `RouteError`, not a blank page
- [ ] Sidebar nav highlights correctly, `/` is `end`
- [ ] Vault lock state is visible in the sidebar
- [ ] No horizontal scroll at 375px width
- [ ] Bottom tabs clear the iOS home indicator
- [ ] Two `<nav>`s have distinct `aria-label`s
- [ ] Keyboard: tab through the whole sidebar in order
- [ ] Dark mode works on every stub screen

---

## Check yourself before Chapter 6

1. **Why build the shell before the vault?**
2. **Why is the unlock gate outside `AppShell` rather than inside it?**
3. **Why does `<main>` need `min-w-0`?**
4. **Which two Settings rows are trust features, and why can they not wait?**
5. **Why is `/documents/:id` a route param and not component state?**
6. **What does the sidebar's lock-state card buy you?**
7. **A user sends you a screenshot showing the sidebar pushed off screen. What CSS is missing?**

---

**Next: [Chapter 6 — The Vault](./06-the-vault.md)** — AES-GCM, PBKDF2, the sealed session,
the profile graph, and `useSyncExternalStore` to bridge the vault into React.