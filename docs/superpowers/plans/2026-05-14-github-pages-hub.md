# GitHub Pages Hub Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Publish all revision tools to `https://nooogle.github.io/EliGCSEs/` with a gothic-styled home page hub and a flashcard app that loads decks via `fetch()` on any browser.

**Architecture:** A new `index.html` at the repo root serves as the hub — it fetches `exams.json` to render a live next-exam countdown and "coming up" list, and links to the existing tools. The flashcard app gains a `fetchMode` that activates on `https:`/`http:` — it loads deck files via `fetch()` from a new `manifest.json` and saves progress to `localStorage` (already the existing fallback path). The File System API path is preserved for local `file://` use.

**Tech stack:** Vanilla HTML/CSS/JS, no build step. Google Fonts (VT323, Special Elite, Oswald). All files served as GitHub Pages static assets.

---

## File map

| Action | Path | Responsibility |
|--------|------|----------------|
| Create | `flashcards/decks/manifest.json` | Ordered list of deck filenames for fetch-based loading |
| Modify | `flashcards/index.html` | Add `fetchMode` + `loadDecksFromManifest()` |
| Create | `index.html` | Gothic home page hub |
| Modify | `.gitignore` | Exclude `.superpowers/` brainstorm artefacts |

---

## Task 1: Create the deck manifest

**Files:**
- Create: `flashcards/decks/manifest.json`

- [ ] **Step 1: Create the manifest**

```json
["macbeth.json", "worlds_and_lives.json", "medicine_in_britain.json"]
```

Write that content to `flashcards/decks/manifest.json`.

- [ ] **Step 2: Verify**

Open `flashcards/decks/manifest.json` in a text editor and confirm it is valid JSON (array of three strings matching the `.json` filenames in `flashcards/decks/`).

- [ ] **Step 3: Commit**

```bash
git add flashcards/decks/manifest.json
git commit -m "Add deck manifest for fetch-mode loading"
```

---

## Task 2: Add fetch mode to the flashcard app

**Files:**
- Modify: `flashcards/index.html`

The flashcard app currently loads decks exclusively via the File System Access API (`showDirectoryPicker`). We add a `fetchMode` flag and a `loadDecksFromManifest()` function. When running on `https:` or `http:`, the app auto-loads from the manifest instead of prompting for a folder.

No changes are needed to `readProgress`, `writeProgress`, or `writeDeck` — they already fall back to `localStorage` when `state.dirHandle` / `state.progressHandle` / `deck.handle` are `null`, which is exactly what fetch mode leaves them as.

- [ ] **Step 1: Add `fetchMode` to the state object**

In `flashcards/index.html`, find the `state` object (line ~619):

```js
const state = {
  dirHandle: null,           // FileSystemDirectoryHandle
  decks: [],                 // [{ filename, handle, data }]
```

Add `fetchMode: false,` as the first property:

```js
const state = {
  fetchMode: false,          // true when running on https:/http: — loads via fetch()
  dirHandle: null,           // FileSystemDirectoryHandle
  decks: [],                 // [{ filename, handle, data }]
```

- [ ] **Step 2: Add `loadDecksFromManifest()` after the `getLsDecks()` function**

The `getLsDecks()` function ends around line 850. Insert the new function immediately after it:

```js
async function loadDecksFromManifest() {
  try {
    const res = await fetch('decks/manifest.json');
    if (!res.ok) throw new Error('manifest.json not found (' + res.status + ')');
    const filenames = await res.json();

    const results = await Promise.all(filenames.map(async filename => {
      try {
        const r = await fetch('decks/' + filename);
        if (!r.ok) return null;
        const data = await r.json();
        if (!data.cards || !data.title) return null;
        return { filename, handle: null, data };
      } catch { return null; }
    }));

    state.decks = results.filter(Boolean);
    renderDeckList();
  } catch (e) {
    document.getElementById('deck-list').innerHTML =
      `<div style="color:var(--text-muted);font-size:.9rem;grid-column:1/-1;text-align:center;padding:40px 0;">
        Could not load decks: ${esc(e.message)}
      </div>`;
  }
}
```

- [ ] **Step 3: Update `init()` to detect protocol and branch into fetch mode**

Find the `init()` IIFE at the bottom of the script (line ~1216):

```js
(function init() {
  if (!state.fsapiSupported) {
    document.getElementById('ls-notice').style.display = 'block';
    document.getElementById('folder-banner').style.display = 'none';
    // Load any localStorage-saved decks
    state.decks = getLsDecks();
    renderDeckList();
  } else {
    const last = localStorage.getItem('fc_last_folder_name');
    if (last) {
      document.getElementById('folder-name').textContent = last + '/';
      document.getElementById('folder-hint').textContent = 'Click "Open Folder" to reload (required each session)';
    }
    renderDeckList();
  }
})();
```

Replace it with:

```js
(function init() {
  state.fetchMode = (window.location.protocol === 'https:' || window.location.protocol === 'http:');

  if (state.fetchMode) {
    document.getElementById('folder-banner').style.display = 'none';
    loadDecksFromManifest();
    return;
  }

  if (!state.fsapiSupported) {
    document.getElementById('ls-notice').style.display = 'block';
    document.getElementById('folder-banner').style.display = 'none';
    state.decks = getLsDecks();
    renderDeckList();
  } else {
    const last = localStorage.getItem('fc_last_folder_name');
    if (last) {
      document.getElementById('folder-name').textContent = last + '/';
      document.getElementById('folder-hint').textContent = 'Click "Open Folder" to reload (required each session)';
    }
    renderDeckList();
  }
})();
```

- [ ] **Step 4: Guard the `ls-notice` in `renderDeckList()`**

In `renderDeckList()` (line ~770), find:

```js
  if (!state.fsapiSupported) {
    document.getElementById('ls-notice').style.display = 'block';
  }
```

Replace with:

```js
  if (!state.fsapiSupported && !state.fetchMode) {
    document.getElementById('ls-notice').style.display = 'block';
  }
```

- [ ] **Step 5: Verify locally with a dev server**

Run a local HTTP server from the repo root (Python is available on macOS):

```bash
python3 -m http.server 8080
```

Open `http://localhost:8080/flashcards/` in Chrome. Expected:
- No "Open Folder" banner visible
- All three decks load automatically (Macbeth, Worlds & Lives, Medicine in Britain)
- Clicking a deck starts a study session
- Progress saves across page refreshes (check via localStorage in DevTools → Application)

Kill the server with `Ctrl-C`. Also open `flashcards/index.html` directly as a `file://` URL and confirm the folder-picker banner still appears (FSAPI mode unchanged).

- [ ] **Step 6: Commit**

```bash
git add flashcards/index.html
git commit -m "Add fetch mode to flashcard app for GitHub Pages"
```

---

## Task 3: Create the home page hub

**Files:**
- Create: `index.html` (repo root)

The home page reuses `eli-revision.html`'s exact CSS variables, fonts, blood-drip SVG, and overlay styles so everything feels cohesive. It fetches `exams.json` for the countdown banner and coming-up list, and fetches `flashcards/decks/manifest.json` for the deck-count badge.

- [ ] **Step 1: Create `index.html`**

Write the following to `index.html` at the repo root:

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<meta name="viewport" content="width=device-width, initial-scale=1.0">
<title>ELI // REVISION HUB</title>
<link href="https://fonts.googleapis.com/css2?family=VT323&family=Special+Elite&family=Oswald:wght@200;400;700&display=swap" rel="stylesheet">
<style>
*,*::before,*::after{box-sizing:border-box;margin:0;padding:0;}
:root{
  --bg:#0a0805;--surface:#110e09;--surface2:#1a1510;
  --border:rgba(180,140,60,0.18);--border2:rgba(180,140,60,0.35);
  --ink:#c8b87a;--ink2:#7a6e50;--ink3:#3d3828;
  --red:#8b1a1a;--red2:#c0392b;
  --fd:'VT323',monospace;--fb:'Special Elite',cursive;--fl:'Oswald',sans-serif;
}
html,body{min-height:100%;background:var(--bg);color:var(--ink);font-family:var(--fb);-webkit-font-smoothing:antialiased;overflow-x:hidden;}
body::before{content:'';position:fixed;inset:0;background-image:url("data:image/svg+xml,%3Csvg viewBox='0 0 200 200' xmlns='http://www.w3.org/2000/svg'%3E%3Cfilter id='n'%3E%3CfeTurbulence type='fractalNoise' baseFrequency='0.85' numOctaves='4' stitchTiles='stitch'/%3E%3C/filter%3E%3Crect width='100%25' height='100%25' filter='url(%23n)' opacity='0.09'/%3E%3C/svg%3E");opacity:0.5;pointer-events:none;z-index:100;}
body::after{content:'';position:fixed;inset:0;background:repeating-linear-gradient(0deg,transparent,transparent 2px,rgba(0,0,0,0.07) 2px,rgba(0,0,0,0.07) 4px);pointer-events:none;z-index:99;}
.page{max-width:500px;margin:0 auto;padding:0 1.5rem 4rem;}
.drip{width:100%;height:44px;display:block;}
.hdr{margin-bottom:1.5rem;position:relative;padding-top:.5rem;}
.eyebrow{font-family:var(--fl);font-size:11px;font-weight:400;letter-spacing:.25em;color:var(--red2);text-transform:uppercase;margin-bottom:6px;}
.t-main{font-family:var(--fd);font-size:80px;line-height:.88;color:var(--ink);letter-spacing:3px;text-shadow:3px 3px 0 var(--red),7px 7px 0 rgba(139,26,26,.25);}
.t-sub{font-family:var(--fl);font-size:12px;font-weight:200;letter-spacing:.3em;color:var(--ink2);text-transform:uppercase;margin-top:10px;border-left:2px solid var(--red);padding-left:10px;}
.skulls{position:absolute;top:8px;right:0;font-size:22px;opacity:.35;animation:pulse 4s ease-in-out infinite;}
@keyframes pulse{0%,100%{opacity:.3;}50%{opacity:.5;}}

/* Next exam banner */
.banner{background:var(--surface);border:.5px solid var(--border);border-top:2px solid var(--red2);padding:14px 16px;margin-bottom:1.5rem;position:relative;display:flex;align-items:center;justify-content:space-between;gap:12px;text-decoration:none;color:inherit;transition:border-top-color .2s;}
.banner:hover{border-top-color:var(--ink);}
.banner-label{position:absolute;top:-8px;left:12px;font-family:var(--fl);font-size:10px;letter-spacing:.15em;color:var(--red2);background:var(--bg);padding:0 6px;text-transform:uppercase;}
.banner-subject{font-family:var(--fl);font-size:13px;font-weight:700;letter-spacing:.1em;color:var(--ink);text-transform:uppercase;}
.banner-paper{font-family:var(--fl);font-size:10px;font-weight:200;letter-spacing:.08em;color:var(--ink2);text-transform:uppercase;margin-top:3px;}
.banner-meta{font-family:var(--fl);font-size:10px;color:var(--ink3);letter-spacing:.05em;text-transform:uppercase;margin-top:4px;}
.banner-right{text-align:right;flex-shrink:0;}
.banner-days{font-family:var(--fd);font-size:52px;line-height:1;color:var(--red2);text-shadow:0 0 18px rgba(192,57,43,.5);}
.banner-unit{font-family:var(--fl);font-size:10px;letter-spacing:.15em;color:var(--ink3);text-transform:uppercase;}

/* Nav tiles */
.nav-grid{display:grid;grid-template-columns:1fr 1fr;gap:8px;margin-bottom:1.5rem;}
.nav-tile{background:var(--surface);border:.5px solid var(--border);border-top:2px solid var(--red);padding:16px 14px 14px;text-decoration:none;color:inherit;display:flex;flex-direction:column;gap:6px;position:relative;transition:border-top-color .2s,background .2s;}
.nav-tile:hover{background:var(--surface2);border-top-color:var(--ink);}
.nav-tile-icon{font-size:22px;opacity:.8;}
.nav-tile-title{font-family:var(--fl);font-size:13px;font-weight:700;letter-spacing:.08em;color:var(--ink);text-transform:uppercase;}
.nav-tile-sub{font-family:var(--fl);font-size:10px;font-weight:200;letter-spacing:.05em;color:var(--ink2);text-transform:uppercase;line-height:1.4;}
.nav-tile-badge{position:absolute;top:10px;right:10px;font-family:var(--fd);font-size:18px;color:var(--red2);opacity:.8;}

/* Divider */
.divider{display:flex;align-items:center;gap:10px;margin:0 0 1.5rem;}
.div-line{flex:1;height:.5px;background:var(--border);}
.div-lbl{font-family:var(--fl);font-size:10px;font-weight:700;letter-spacing:.22em;color:var(--red2);text-transform:uppercase;}

/* Upcoming list */
.upcoming{display:flex;flex-direction:column;gap:6px;margin-bottom:1.5rem;}
.upcoming-row{background:var(--surface);border:.5px solid var(--border);padding:10px 14px;display:flex;align-items:center;justify-content:space-between;text-decoration:none;color:inherit;transition:border-color .15s;}
.upcoming-row:hover{border-color:var(--border2);}
.upcoming-name{font-family:var(--fl);font-size:11px;letter-spacing:.06em;color:var(--ink2);text-transform:uppercase;}
.upcoming-date{font-family:var(--fl);font-size:10px;letter-spacing:.05em;color:var(--ink3);text-transform:uppercase;white-space:nowrap;}

/* Mantra */
.mantra{text-align:center;margin-top:2rem;}
.mtext{font-family:var(--fd);font-size:30px;color:var(--ink3);letter-spacing:5px;text-transform:uppercase;animation:flicker 7s ease-in-out infinite;}
@keyframes flicker{0%,88%,100%{opacity:.4;}90%,96%{opacity:.1;}93%{opacity:.5;}}
.msub{font-family:var(--fl);font-size:10px;color:var(--ink3);letter-spacing:.22em;text-transform:uppercase;margin-top:4px;opacity:.4;}
</style>
</head>
<body>
<div class="page">

<svg class="drip" viewBox="0 0 500 44" preserveAspectRatio="none" aria-hidden="true">
  <path d="M0,0 L500,0 L500,8 Q450,8 442,20 Q435,32 432,40 Q429,32 426,20 Q418,8 320,8 Q275,8 271,22 Q267,36 265,42 Q263,36 261,22 Q257,8 150,8 Q95,8 89,18 Q83,28 80,36 Q77,28 74,18 Q68,8 0,8Z" fill="#8b1a1a" opacity="0.75"/>
  <path d="M0,0 L500,0 L500,5 Q430,5 423,15 Q417,24 414,31 Q411,24 408,15 Q401,5 290,5 Q248,5 244,17 Q240,28 238,34 Q236,28 234,17 Q230,5 105,5 Q58,5 52,16 Q47,26 44,33 Q41,26 38,16 Q32,5 0,5Z" fill="#c0392b" opacity="0.35"/>
</svg>

<div class="hdr">
  <div class="eyebrow">&#9760; gcse 2026 &#9760;</div>
  <div class="t-main">ELI</div>
  <div class="t-sub">droitwich spa high school &mdash; revision hub</div>
  <div class="skulls">&#9760;&#9760;</div>
</div>

<a class="banner" id="banner" href="calendar.html" style="display:none">
  <span class="banner-label">Next Exam</span>
  <div>
    <div class="banner-subject" id="banner-subject"></div>
    <div class="banner-paper" id="banner-paper"></div>
    <div class="banner-meta" id="banner-meta"></div>
  </div>
  <div class="banner-right">
    <div class="banner-days" id="banner-days"></div>
    <div class="banner-unit" id="banner-unit"></div>
  </div>
</a>

<div class="nav-grid">
  <a class="nav-tile" href="flashcards/index.html">
    <div class="nav-tile-icon">🃏</div>
    <div class="nav-tile-title">Flashcards</div>
    <div class="nav-tile-sub" id="tile-decks">Spaced repetition</div>
    <div class="nav-tile-badge" id="badge-decks"></div>
  </a>
  <a class="nav-tile" href="calendar.html">
    <div class="nav-tile-icon">📅</div>
    <div class="nav-tile-title">Timetable</div>
    <div class="nav-tile-sub" id="tile-exams">Exam countdown</div>
    <div class="nav-tile-badge" id="badge-exams"></div>
  </a>
  <a class="nav-tile" href="eli-revision.html">
    <div class="nav-tile-icon">⏱</div>
    <div class="nav-tile-title">Timer</div>
    <div class="nav-tile-sub">Pomodoro &middot; session log</div>
  </a>
  <a class="nav-tile" href="#">
    <div class="nav-tile-icon">📚</div>
    <div class="nav-tile-title">Resources</div>
    <div class="nav-tile-sub">Coming soon</div>
  </a>
</div>

<div class="divider" id="divider-upcoming" style="display:none">
  <div class="div-line"></div>
  <div class="div-lbl">&#9760; coming up &#9760;</div>
  <div class="div-line"></div>
</div>

<div class="upcoming" id="upcoming-list"></div>

<div class="mantra">
  <div class="mtext">MEMENTO MORI</div>
  <div class="msub">the exams are coming</div>
</div>

</div>
<script>
const DAYS = ['Sun','Mon','Tue','Wed','Thu','Fri','Sat'];
const MONTHS = ['Jan','Feb','Mar','Apr','May','Jun','Jul','Aug','Sep','Oct','Nov','Dec'];

function todayStr() {
  return new Date().toISOString().split('T')[0];
}

function formatDate(dateStr) {
  const d = new Date(dateStr + 'T00:00:00');
  return DAYS[d.getDay()] + ' ' + d.getDate() + ' ' + MONTHS[d.getMonth()];
}

function daysUntil(dateStr) {
  const today = new Date(todayStr() + 'T00:00:00');
  const exam  = new Date(dateStr  + 'T00:00:00');
  return Math.round((exam - today) / 86400000);
}

async function loadExams() {
  try {
    const res = await fetch('exams.json');
    if (!res.ok) return;
    const data = await res.json();
    const today = todayStr();
    const upcoming = data.exams
      .filter(e => e.date >= today)
      .sort((a, b) => a.date.localeCompare(b.date) || a.time.localeCompare(b.time));

    if (!upcoming.length) return;

    // Banner — first upcoming exam
    const next = upcoming[0];
    const days = daysUntil(next.date);
    document.getElementById('banner-subject').textContent = next.subject;
    document.getElementById('banner-paper').textContent   = next.paper + ' · ' + next.code;
    document.getElementById('banner-meta').textContent    =
      formatDate(next.date) + ' · ' + next.time + ' · ' + next.location + ' · Seat ' + next.seat;
    document.getElementById('banner-days').textContent    = days === 0 ? 'Today' : days;
    document.getElementById('banner-unit').textContent    =
      days === 0 ? '' : days === 1 ? 'day to go' : 'days to go';
    document.getElementById('banner').style.display = 'flex';

    // Exam count badge
    document.getElementById('badge-exams').textContent = data.exams.length;
    document.getElementById('tile-exams').textContent  = data.exams.length + ' exams total';

    // Coming up — next 4 after the banner exam
    const rest = upcoming.slice(1, 5);
    if (rest.length) {
      const list = document.getElementById('upcoming-list');
      rest.forEach(e => {
        const a = document.createElement('a');
        a.className = 'upcoming-row';
        a.href = 'calendar.html';
        a.innerHTML =
          '<span class="upcoming-name">' + esc(e.subject) + ' &mdash; ' + esc(e.paper) + '</span>' +
          '<span class="upcoming-date">' + esc(formatDate(e.date)) + '</span>';
        list.appendChild(a);
      });
      document.getElementById('divider-upcoming').style.display = 'flex';
    }
  } catch { /* silently skip if fetch fails */ }
}

async function loadDeckCount() {
  try {
    const res = await fetch('flashcards/decks/manifest.json');
    if (!res.ok) return;
    const filenames = await res.json();
    document.getElementById('badge-decks').textContent = filenames.length;
    document.getElementById('tile-decks').textContent  = filenames.length + ' decks · spaced repetition';
  } catch { /* silently skip */ }
}

function esc(str) {
  return String(str)
    .replace(/&/g,'&amp;').replace(/</g,'&lt;')
    .replace(/>/g,'&gt;').replace(/"/g,'&quot;');
}

loadExams();
loadDeckCount();
</script>
</body>
</html>
```

- [ ] **Step 2: Verify locally**

Serve from the repo root:

```bash
python3 -m http.server 8080
```

Open `http://localhost:8080/` in Chrome. Verify:
- Blood-drip header and gothic fonts render
- Next-exam banner appears with correct subject, paper, date, seat, and day count
- Flashcards tile shows deck count (e.g. "3")
- Timetable tile shows exam count (e.g. "19")
- Coming-up list shows the next 4 exams after the banner exam
- All 4 tile links resolve without 404 (Timer → `eli-revision.html`, Timetable → `calendar.html`, Flashcards → `flashcards/index.html`, Resources → `#`)
- Clicking Flashcards tile opens the app and auto-loads all decks

Kill the server with `Ctrl-C`.

- [ ] **Step 3: Commit**

```bash
git add index.html
git commit -m "Add gothic home page hub with exam countdown"
```

---

## Task 4: Tidy up and enable GitHub Pages

**Files:**
- Modify: `.gitignore`

- [ ] **Step 1: Add `.superpowers/` to `.gitignore`**

Open `.gitignore`. Append:

```
.superpowers/
```

- [ ] **Step 2: Commit**

```bash
git add .gitignore
git commit -m "Ignore .superpowers/ brainstorm artefacts"
```

- [ ] **Step 3: Push to GitHub**

```bash
git push origin master
```

Expected: push succeeds, no errors.

- [ ] **Step 4: Enable GitHub Pages in repo settings**

1. Go to `https://github.com/nooogle/EliGCSEs/settings/pages`
2. Under **Source**, select **Deploy from a branch**
3. Branch: `master`, folder: `/ (root)`
4. Click **Save**

GitHub will show a banner: _"Your site is being published at https://nooogle.github.io/EliGCSEs/"_. It typically takes 1–2 minutes.

- [ ] **Step 5: Verify on GitHub Pages**

Open `https://nooogle.github.io/EliGCSEs/` in any browser (Chrome, Firefox, Safari, mobile). Verify:
- Home page loads with gothic styling and live countdown banner
- Clicking Flashcards opens `https://nooogle.github.io/EliGCSEs/flashcards/` and auto-loads all three decks without any folder-picker prompt
- Study a card and rate it; reload the page and confirm the card is not shown as due (progress persisted to localStorage)
- Check on a mobile browser — layout should be readable at 375px width
