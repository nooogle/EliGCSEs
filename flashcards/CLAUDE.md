# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Running the app

Open `index.html` directly in a browser — no build step, no server required. For full File System Access API support (read/write to the `decks/` folder), use Chrome or Edge. Firefox falls back to localStorage automatically.

## Architecture

Everything lives in a single `index.html` file with three logical sections:

**HTML** — three `<div class="screen">` panels (home, study, progress) plus one modal (add card). Only one screen has `class="active"` at a time; the topbar is shared.

**CSS** — CSS custom properties for the design system (`--accent`, `--green`, `--amber`, `--red`, etc.). The flashcard flip uses `transform-style: preserve-3d` with two `.flashcard-face` elements (`.front` / `.back`); toggling `.flipped` on the wrapper triggers the CSS rotation.

**JavaScript** — all in a single `<script>` tag, no modules. Key sections:

- `state` object — single source of truth: `dirHandle`, `decks[]`, `activeDeck`, `progress`, `queue[]`, `session` stats.
- `sm2(record, quality)` — pure function implementing the SM-2 algorithm. Returns updated `{ easiness, interval_days, repetitions, next_due }`.
- File I/O — `readProgress` / `writeProgress` / `writeDeck` attempt the File System Access API first, then fall back to localStorage. Both paths are always kept in sync (LS is always written).
- Screen routing — `showScreen(name)` switches active screen; `setTopbar(title, actions)` rebuilds the topbar buttons for context.
- `startStudy(deck)` — loads progress, filters due cards, shuffles into `state.queue`, then calls `renderStudyCard()` in a loop via user responses.

## Deck format

JSON files in `decks/` with `.json` extension (not `.progress.json`). Required fields: `title`, `subject`, `cards[]`. Each card needs `id`, `question`, `answer`; optional: `category`, `hint`.

Progress is stored in sidecar files: `decks/macbeth.json` → `decks/macbeth.progress.json`. Schema per card entry: `{ easiness, interval_days, repetitions, next_due, history[] }`.

## SM-2 quality mapping

| Button   | Quality | Behaviour                        |
|----------|---------|----------------------------------|
| Got it!  | 5       | Normal interval increase         |
| Nearly   | 3       | Smaller increase, EF preserved   |
| No idea  | 1       | Reset reps to 0, due tomorrow    |

Cards are "mastered" when `repetitions >= 3`.

## Keyboard shortcuts (study screen)

- `Space` / `Enter` — flip card
- `1` — No idea, `2` — Nearly, `3` — Got it (after flip)
