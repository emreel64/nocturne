# Nocturne — Project Handoff

This is the website for **Nocturne**, a student magazine from Özel İzmir Amerikan Koleji (ACI). It's a static site hosted on Netlify and connected to a GitHub repo. The owner is a high schooler running the magazine — code clarity matters more than cleverness.

Read this whole file before making changes. The architecture has a few non-obvious conventions that will trip you up if you skip ahead.

---

## What the site does

Two pages, both styled as a single editorial brand:

1. **`index.html`** — landing page. Hero with sunset photo + interactive cursor glow + 3D pill buttons. Editorial intro section. Issue gallery. Footer with newsletter subscribe form. Self-contained About + Join Us + Issues-picker modals.

2. **`read.html`** — the magazine reader. Renders PDFs page-by-page using PDF.js + StPageFlip as a "flipbook". Has Prev/Next, jump-to-page, magnifier (loupe), enlarge (whole-magazine zoom via CSS `zoom`), download dropdown, issues drawer, Join Us drawer, About drawer, and an onboarding tour (one-time, remembered via localStorage).

The two pages share a brand system and link to each other. The landing's "Read" button goes to `read.html?open=issues`, which makes the reader skip its initial render and pop the issues drawer first — visitor picks an issue, *then* the magazine renders.

---

## File layout

```
nocturne-deploy/                      ← this is the deploy root + GitHub repo source
├── index.html                        ← landing page
├── read.html                         ← magazine reader
├── favicon.svg                       ← pink "N" on dark
├── content/
│   └── issues.json                   ← canonical issue list, edited via /admin/ CMS
├── admin/
│   ├── index.html                    ← Decap CMS bootstrap (tiny, ~50 lines)
│   └── config.yml                    ← CMS field/collection config
├── covers/
│   ├── nocturne-2024.jpg             ← static cover thumbnails (≈200KB)
│   └── nocturne-2025.jpg
├── assets/
│   └── sunset.jpg                    ← hero background, ≈220KB
├── Nocturne.pdf                      ← Issue 01 source (≈8MB, compressed)
└── nocturne2_final12.pdf             ← Issue 02 source (≈6.5MB, compressed)
```

There is also a **working copy** at `/Users/emre/Downloads/magazine.html` and `/Users/emre/Downloads/index.html`. The user historically edited there, then synced to `nocturne-deploy/` via:

```bash
cp Downloads/magazine.html Desktop/nocturne-deploy/read.html
sed 's/magazine\.html/read.html/g' Downloads/index.html > Desktop/nocturne-deploy/index.html
```

You don't have to follow that pattern — feel free to edit `nocturne-deploy/` files directly. Just note that the working-copy filename is `magazine.html` while the deployed filename is `read.html`. **All internal links in `index.html` should point to `read.html`** (the deploy filename).

---

## Tech stack — what's used and what isn't

- **Plain HTML / CSS / vanilla JS.** No build step. No bundler. No framework.
- **PDF.js 3.11.174** (CDN) — renders PDF pages to canvas → JPEG data URLs.
- **StPageFlip 2.0.7** (CDN) — the flipbook library. Receives the JPEG list and handles page-flipping animation.
- **Decap CMS 3.x** (CDN) — the editor at `/admin/`. Git-Gateway backend.
- **Netlify Identity widget** (CDN) — auth for the CMS. Loaded on every page so invite-token URLs work anywhere.
- **FormSubmit** (`https://formsubmit.co/ajax/nocturneaci@gmail.com`) — handles the Join Us form. Sends to that Gmail; subject-line differentiates form types.
- **Netlify Forms** — handles the Subscribe form. Form is statically declared in `index.html` with `data-netlify="true"`.
- **localStorage** — used for: subscriber dedup (key `nocturne-subscribed-emails-v1`), tour completion (`nocturne-tour-completed-v1`), and forced logout from `/admin/`.

There's a Claude Design handoff bundle that defines the brand system in React JSX. **Do not port this to React.** The bundle is reference-only; the live site is vanilla JS. The CSS/colors/typography from `colors_and_type.css` in that bundle are already inlined into `index.html` and `read.html`.

---

## Brand system (don't drift from this)

```
--ink:           #1a1a1a   /* near-black text on paper */
--paper:         #f4efe6   /* warm cream */
--paper-shadow:  #d9d0bf   /* darker cream */
--accent:        #2b4a6f   /* navy — primary CTA, buttons, focus rings */
                            /* (NOT the original orange — that was replaced) */
--bg-1:          #2a2622   /* warm charcoal */
--bg-2:          #161412   /* near-black, bg of dark surfaces */
--muted:         #8a8275

PINK WORDMARK COLOR: #f5c2d8 (pastel pink, used for "Nocturne" everywhere)
```

Fonts: **Archivo Black** for the wordmark/display, **Fraunces** (serif) for editorial body, **Inter** for UI/forms/buttons.

---

## Issue data flow (CMS-managed)

The single source of truth for issues is **`content/issues.json`**:

```json
{
  "current_issue_id": "nocturne-2025",
  "issues": [
    { "id": "...", "label": "...", "title": "Nocturne", "file": "*.pdf", "cover": "covers/*.jpg" }
  ]
}
```

**Both pages fetch this on load and render dynamically.** If the fetch fails (offline / opening from `file://`), each page has a `FALLBACK_ISSUES` hardcoded array that mirrors the current JSON — so local previews don't break.

- `read.html` populates its `ISSUES` JS array from the fetch, then runs `init()`.
- `index.html` calls `renderIssueData(data)` to update the hero eyebrow/cover, the topbar Issues dropdown, the picker modal grid, and the gallery cards.

When an editor adds Issue 03 via `/admin/`, Decap commits an updated `content/issues.json` plus the new PDF/cover to git, Netlify rebuilds, and the new issue appears on next page load. **You don't need to touch HTML to add a new issue.**

---

## Deep-link URL params (read.html)

The reader honors these query params:

- `?issue=<id>` — load that issue instead of the latest
- `?open=issues` — **special case**: SKIP the initial render entirely, hide the loading screen, pop the issues drawer immediately. Only after the user picks an issue does loading start. This is what the landing's "Read" button uses.
- `?open=join` — load the latest issue, then pop the Join Us drawer
- `?open=about` — load the latest, pop the About drawer

The `?open=issues` flow also defers the onboarding tour: it waits until the user picks from the drawer, *then* fires the tour after that issue has rendered. See `deferTourUntilIssueSelect` and `fireDeferredTour`.

---

## How the reader actually renders pages

1. `loadIssue(issue)` calls `renderPdfToImages(issue.file, onProgress)`.
2. PDF.js renders **two JPEGs per page**:
   - `images[i]` — display-sized (scale 1.6×dpr, capped 2.0–4.0). Used for the in-page `<img>` so the browser barely has to downsample.
   - `zoomImages[i]` — high-res (scale 4.0). Used by the magnifier for near-1:1 sampling.
3. The display JPEG is created by drawing the hi-res canvas onto a smaller canvas with `imageSmoothingQuality: 'high'`, *then* `toDataURL('image/jpeg', 0.95)`. This avoids the browser's default downsample which made text look soft.
4. PDF.js render uses `intent: 'print'` for the highest-quality path.
5. Both image lists go into `issueCache[issue.id]` so switching issues doesn't re-render.
6. `buildBook(images, aspect)` initializes StPageFlip with those images.

Don't change the dual-rendering setup unless you have a specific reason — the user spent significant time getting magnifier + display sharpness to match.

---

## The 3D pill buttons

Four buttons on the landing page are wrapped as 3D pills you can press-and-drag to rotate:

- Hero "Read" (primary)
- Hero "Open the latest issue" (secondary)
- Gallery "Open the reader" (secondary)
- Footer "Subscribe"

Implementation lives in **`index.html`**, around `setupTiltBtn` / `build3DPill`. On init, JS replaces each `.tilt-btn`'s content with a `<span class="tilt-stage">` containing:

- 4 perpendicular side panels (`.tilt-edge-top/bottom/left/right`) — flat rectangles rotated 90° around X/Y to face outward
- A back face (`.tilt-face--back`) — color-flipped (pink bg, navy text), `translateZ(-8px) rotateY(180deg)`
- A front face (`.tilt-face--front`) — re-creates the original button look in normal flow so it sizes the stage. `translateZ(8px)`.

Press → drag rotates the stage (rotateX from drag-Y, rotateY from drag-X, scaled 1.5×). Release → empty the inline transform → CSS transition (`cubic-bezier(0.34, 1.56, 0.64, 1)`) bounces it home.

**Known visual imperfection:** the front/back have `border-radius: 999px` (full pill) but the side panels are rectangular — at the rounded ends the rectangular sides don't perfectly meet the curved corners. CSS doesn't have native curved 3D surfaces. To fix this perfectly you'd need ~24 small flat panels arrayed around each rounded end (a faceted approximation of the curve). The user is aware of this trade-off.

---

## Onboarding tour

Skippable guided walkthrough. Auto-runs once per browser (remembered via `nocturne-tour-completed-v1` in localStorage). Re-triggerable from the "Take a tour" link inside the About drawer.

8 steps: welcome → Issues → Download → Join Us → About → reading controls → Enlarge → Magnifier. Each step dims the page, draws a soft accent ring around the target, shows an orange popup with a tail that slides smoothly between targets.

The tour code is large but self-contained inside `init()`. Search for `TOUR_STEPS`.

---

## Auth & CMS workflow

- `/admin/` loads Decap CMS, which wraps Netlify Identity for auth.
- `admin/index.html` does a **logout-on-arrival**: it wipes `localStorage.gotrue.user` and `nf_jwt` cookie *before* Netlify Identity loads, so editors always have to re-enter their password.
- The footer "Login" link on `index.html` does the same logout-then-login flow via the Identity widget popup, *then* redirects to `/admin/`.
- A custom show/hide password toggle is injected into the Netlify Identity widget DOM (the widget renders into the page, not an iframe — so we can find its `input[type="password"]` and add an eye button).
- Editors are invite-only — invited via Netlify dashboard → Identity → Invite users.

If editors complain that the invite link goes to the homepage and nothing happens, it means the Netlify Identity widget script didn't load on `index.html`. Check the `<head>`.

---

## Local development

Open `Downloads/index.html` (or `nocturne-deploy/index.html`) directly in a browser via `file://`. Most things work, but:

- **Netlify Forms** (subscribe) won't work locally — they need the live Netlify backend. Local submissions silently hang or fail.
- **`fetch('content/issues.json')`** may fail under `file://` (CORS-ish). The fallback hardcoded data kicks in, so the page still renders both issues.
- **Decap CMS** at `/admin/` won't work locally without `local_backend: true` in `config.yml` and running `npx decap-server`. Easier to test CMS changes on a Netlify deploy preview.
- **PDF.js rendering** does work locally if the PDFs are in the same folder.

For meaningful end-to-end testing, push to GitHub and let Netlify rebuild — the deploy preview URL is the realistic environment.

---

## Deployment

Netlify is connected to the GitHub repo (continuous deployment). Pushing to the `main` branch triggers a rebuild. There is no build command — Netlify just publishes the repo root as the site.

The user originally used drag-and-drop deploys (zip the folder, drag onto Netlify) before Git was connected. Both still work; Git is canonical now.

The user's billing is on the free Starter plan (300 credits/month). The `_headers` file is **not yet added** but should be — it would let visitors cache PDFs for a year and significantly cut bandwidth credits. Suggested content:

```
/Nocturne.pdf
  Cache-Control: public, max-age=31536000, immutable
/nocturne2_final12.pdf
  Cache-Control: public, max-age=31536000, immutable
/covers/*
  Cache-Control: public, max-age=31536000, immutable
/assets/*
  Cache-Control: public, max-age=31536000, immutable
```

---

## Things in flight / loose ends

- `admin/config.yml` has `site_url: https://nocturne-aci.netlify.app` as a placeholder. Update if the actual domain is different.
- Subscribe dedup is **client-side only** (localStorage). Cross-browser duplicates aren't caught. Real solution would be Buttondown or similar — user has been told this is acceptable for now.
- The Join Us form still uses **FormSubmit**, not Netlify Forms. Subscribe was migrated; Join wasn't, because it works fine. Could consolidate.
- 3D pill button sides are rectangular (see "3D pill buttons" section above). Refining to curved arcs is a known follow-up.
- The Decap CMS edits `content/issues.json`. New PDFs uploaded via the CMS go to `/uploads/` (set in `config.yml`). Existing PDFs (`Nocturne.pdf`, `nocturne2_final12.pdf`) are at the root; this is fine — the JSON just stores whatever path each issue actually lives at.

---

## Communication style

The user is a high-schooler, not a developer. Don't dump jargon-heavy explanations. When you make a change, briefly say *what* changed in plain language and *what they should do next* (push to GitHub, test the live URL, etc.). Don't drown them in code unless they ask to see it.

When they describe something in casual English with typos, charitably interpret what they meant. They're describing UX in their own words.

---

## Quick reference: where to find common things

| What | Where |
|---|---|
| Brand colors | CSS `:root` at top of `index.html` and `read.html` |
| Issue list | `content/issues.json` (CMS-edited) |
| Hero markup | `index.html` → `<section class="hero">` |
| Cursor glow + ripples | `index.html` JS → `// Hero — cursor glow + click ripples` |
| 3D pill button code | `index.html` JS → `setupTiltBtn` / `build3DPill` |
| Subscribe form | `index.html` HTML around `id="subscribeForm"`, JS around `// Subscribe form` |
| Join form | `index.html` modal + `read.html` drawer (duplicated structure, same FormSubmit endpoint) |
| Country code dropdown | Same place as Join form, look for `COUNTRIES = [` |
| Tour | `read.html` → `TOUR_STEPS = [` |
| Magnifier | `read.html` → `// Magnifier (loupe)` |
| PDF rendering | `read.html` → `function renderPdfToImages` |
| Admin/CMS | `admin/index.html`, `admin/config.yml` |

Good luck.
