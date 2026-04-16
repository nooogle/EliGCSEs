# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Purpose

This project contains study tools to help Eli (Elise Shadforth) prepare for her GCSEs at Droitwich Spa High School. Exams run 11 May – 15 Jun 2026.

## Eli's GCSE Subjects

| Subject | Board | Notes |
|---------|-------|-------|
| English Literature | AQA 8702 | Macbeth; Frankenstein; An Inspector Calls; Worlds & Lives poetry |
| English Language | AQA 8700 | 2 papers |
| Religious Studies | AQA 8062 | Option Oa (Christianity/Sikhism) |
| Combined Science: Trilogy | AQA 8464 | Biology, Chemistry, Physics — Foundation tier |
| Mathematics | Pearson/Edexcel 1MA1 | Foundation tier, 3 papers |
| History | Pearson/Edexcel 1HI0 | Medicine in Britain; Superpower Relations; Weimar & Nazi Germany |
| Design and Technology | AQA 8552 | Written paper only |

## Key Data Files

| Path | Description |
|------|-------------|
| `exams.json` | Full exam timetable — canonical source of truth |
| `exams.ics` | iCalendar file for import into phone/Google calendar |
| `priorities.md` | College entry goal, subject priorities, predictions, revision tracker |

## Structure

| Path | Description |
|------|-------------|
| `flashcards/` | HTML5 flashcard SPA using SM-2 spaced repetition |
| `flashcards/index.html` | Complete single-file app — open directly in Chrome/Edge |
| `flashcards/decks/` | JSON deck files + sidecar `.progress.json` files |
| `flashcards/CLAUDE.md` | Flashcard app architecture and technical reference |
| `exam_reference/` | Spec details, paper structure, AOs, set texts per subject |
| `resources/` | Online revision links and quizzes, one file per subject |

Each tool subdirectory has its own `CLAUDE.md` with app-specific guidance — read it before making changes to that tool.

## Adding a resource link

Add URLs to `resources/<subject>.md`. Create the file if it doesn't exist yet, following the same format as `resources/english_literature.md`. One line per link: `- [Title](URL) — brief note`.

## Project Conventions

- **GitHub Issues** are used for todos and reminders — not local todo lists.
