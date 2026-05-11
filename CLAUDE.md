# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**Musculoskelefun!** is a single-file educational PWA for Primary 2 kids (~age 8) on iPad Air M2. The entire app lives in `index.html` — there is no build step, no package manager, and no test suite. All changes are made directly to `index.html`.

Target deployment: Vercel (static) or GitHub Pages. `vercel.json` is already configured.

## Deployment

```bash
# Push to feature branch (main development branch)
git push -u origin claude/access-human-bone-project-iLnWH

# main branch on GitHub may lag behind — sync it by pushing files via
# the GitHub MCP tool (mcp__github__push_files) targeting branch "main",
# since direct git push to main returns HTTP 403.
```

To preview locally: open `index.html` directly in a browser (`file://` works — no server needed).

## Architecture

The app is **one HTML file** (~2500 lines) with inline CSS and inline JS, no external dependencies except:
- Google Fonts (Baloo 2 + Nunito) — loaded in `<head>`
- Three.js r128 — loaded as the **last `<script>` tag** in `<body>` (intentionally non-blocking so the quiz works before 3D is ready)

### Screen System

Five `.screen` siblings in `<body>`. Only one has `.active` at a time, toggled by `showScreen(id)`:

| Screen id | Description |
|-----------|-------------|
| `#home`   | Chapter card grid + explorer card + quiz button |
| `#lesson` | Lesson pages with dot navigation |
| `#quiz`   | Quiz questions one at a time |
| `#result` | Score, animations, gift boxes |
| `#explorer` | Full-screen Three.js 3D skeleton |

`#explorer.screen.active` uses `position: fixed` (not the normal flex layout) to overlay the full viewport without touching `body` overflow — this is the key trick that lets the scrollable quiz screens coexist with the full-screen 3D canvas.

### CSS Architecture

- App1 (quiz/lesson) variables are on `:root`
- App2 (explorer) dark-theme variables are scoped inside `#explorer {}` blocks
- CSS sections are labelled with `/* ===== SECTION NAME ===== */` comments
- **Do not use en-dashes (`–`) in CSS property names** — a previous version had `var(–dark)` instead of `var(--dark)` which broke all custom properties. Always use double hyphens.

### JS Architecture — Two Independent Systems

**Quiz/Lesson system** (lines ~1274–1780):
- `lessons[]` — array of 4 lesson objects, each with a `pages[]` array of HTML strings
- `questions[]` — 15 static question objects; shuffled into `activeQuestions[]` at quiz start via Fisher-Yates `shuffleArray()`
- State: `completedLessons` (Set), `visitedLessonPages` (object of Sets), `currentLesson`, `currentLessonPage`, `currentQuestion`, `score`, `answered`
- `updateHomeScreen()` — call this whenever lesson progress changes; it updates card progress bars, CSS classes, and quiz button text

**3D Explorer system** (lines ~1782–2472):
- Lazy-initialised: `initExplorer()` only runs on first `goExplorer()` call; guarded by `expReady` flag
- RAF loop uses two flags: `expRunning` (set false in `goHome()`) and `expRafActive` (prevents duplicate loop starts). Always call `goHome()` to leave the explorer — never just `showScreen()`.
- `boneMeshes` — object mapping bone id → array of Three.js Mesh objects
- `BONES[]` — 12 bone data objects with id, name, latin, emoji, desc, fun, group
- Touch events are bound to the **canvas element**, not `document`/`window`, to avoid eating quiz scroll events

### Key Flows

**Adding a lesson**: Add a new object to `lessons[]`. Update `renderLesson()` if the last-lesson logic needs to change.

**Adding a quiz question**: Add to `questions[]` — it is shuffled automatically. Update `activeQuestions.length` references if the total changes (currently hardcoded display says "15" in the question counter HTML but uses `activeQuestions.length` in JS).

**Adding a bone to the 3D model**: Add to `BONES[]` and add geometry calls in `buildSkeleton()`. The bone list sidebar is auto-generated from `BONES[]` by `buildBoneList()`.

**Result screen score tiers**: `showResult()` branches on `score === total` (perfect), `pct >= 0.8`, `pct >= 0.6`, `pct >= 0.4`, and catch-all. Gift boxes are always rendered in DOM but have `.locked` class removed only on perfect score.

### PWA Icon

The `<link rel="apple-touch-icon">` href is populated at runtime by an IIFE that draws a 512×512 canvas and injects the PNG data-URL. The static `icon.svg` is used by `manifest.json` as a fallback. The manifest itself is also replaced at runtime with one pointing to the canvas-generated PNG.
