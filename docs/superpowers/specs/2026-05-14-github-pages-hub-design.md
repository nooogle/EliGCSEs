# Design: GitHub Pages Hub

**Date:** 2026-05-14  
**Repo:** nooogle/EliGCSEs  
**Published URL:** https://nooogle.github.io/EliGCSEs/

---

## Goal

Make the revision tools accessible from any browser (phone, school computer, tablet) without needing to open local files. A home page hub at the repo root ties all tools together. The flashcard app is adapted to load decks via `fetch()` instead of the File System Access API.

---

## Architecture

### New files

| File | Purpose |
|------|---------|
| `index.html` | Home page hub — gothic aesthetic, exam countdown, nav tiles |
| `flashcards/decks/manifest.json` | Ordered list of deck filenames for fetch-based loading |

### Modified files

| File | Change |
|------|--------|
| `flashcards/index.html` | Add fetch-mode deck loading; auto-select mode by protocol |

### GitHub Pages config

Enable in repo settings: **Settings → Pages → Source: Deploy from branch → `master` → `/ (root)`**.  
No custom domain. URL: `https://nooogle.github.io/EliGCSEs/`

---

## Home Page (`index.html`)

### Visual design

Identical aesthetic to `eli-revision.html`:
- CSS variables: `--bg:#0a0805`, `--red:#8b1a1a`, `--ink:#c8b87a`, etc.
- Fonts: VT323 (display), Special Elite (body), Oswald (labels) via Google Fonts
- Body overlays: SVG noise grain + scanline gradient (both `position:fixed`, `pointer-events:none`)
- Blood-drip SVG at top (copied from `eli-revision.html`)
- `max-width: 500px`, centred

### Dynamic data

On `DOMContentLoaded`, fetch `exams.json` (relative path, works on GitHub Pages and locally). Parse and:

1. **Next exam banner** — find the first exam where `date >= today` (compare YYYY-MM-DD strings). Display subject, paper code, date formatted as "Fri 15 May", time, location, seat number. Days remaining: `Math.ceil((examDate - today) / 86400000)`. If 0 days: show "Today". If 1: "Tomorrow". Else: "N days to go". Banner links to `calendar.html`.

2. **Coming up list** — the next 4 exams from the same sorted list (starting from the exam after the banner exam, or including the banner exam if < 4 total remain). Each row links to `calendar.html`.

If `fetch('exams.json')` fails (unlikely on GitHub Pages, possible locally without a server), hide both dynamic sections gracefully — no broken UI.

### Navigation tiles

2×2 grid, gothic panel style (dark surface, red top border, hover state):

| Tile | Links to | Badge |
|------|----------|-------|
| Flashcards 🃏 | `flashcards/index.html` | Deck count — length of manifest array (e.g. "3 decks") |
| Timetable 📅 | `calendar.html` | Total exam count from exams.json (e.g. "19 exams") |
| Timer ⏱ | `eli-revision.html` | — |
| Resources 📚 | `#` (placeholder, not yet implemented) | — |

Both badges come from data already fetched for other purposes (manifest for deck list, exams.json for countdown). No extra requests needed.

### Static content

- Tagline: "droitwich spa high school — revision hub"
- Footer mantra: "MEMENTO MORI / the exams are coming" (matches eli-revision.html tone)

---

## Flashcard App — Fetch Mode (`flashcards/index.html`)

### Protocol detection

On load, check `window.location.protocol`:

- `'https:'` → **fetch mode** (GitHub Pages or any web server)
- `'file:'` → **File System API mode** (existing local behaviour, unchanged)
- `'http:'` → **fetch mode** (local dev server)

### Fetch mode behaviour

1. Fetch `decks/manifest.json` — array of filenames e.g. `["macbeth.json", "worlds_and_lives.json", "medicine_in_britain.json"]`
2. Fetch each deck file: `decks/<filename>`
3. Populate `state.decks[]` exactly as the File System API path does today
4. Load progress from `localStorage` (key pattern: `progress_<deckname>`) — this is already the existing fallback, no new logic needed
5. Hide or disable the "Open folder" / directory picker UI element — it is irrelevant in fetch mode
6. The "Add card" button writes to localStorage only; the card persists for that browser session. This is acceptable behaviour — new decks are added by committing to the repo.

### manifest.json

```json
["macbeth.json", "worlds_and_lives.json", "medicine_in_britain.json"]
```

Maintaining the manifest is a manual step when adding new decks (add file + add entry to manifest). This is acceptable given the low frequency of deck additions.

### Unchanged behaviour

- SM-2 algorithm, study flow, progress screen — no changes
- File System API path for `file://` — no changes
- localStorage progress persistence — already implemented, no changes

---

## Out of scope

- Resources page (placeholder tile only)
- Custom domain
- Automatic manifest generation
- Service worker / offline support
- Deck creation/editing on GitHub Pages (localStorage-only for added cards)
