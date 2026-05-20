# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A single-file, self-contained web app: **Árealimpa — Avaliação Semanal** (Weekly Evaluation). It's a Portuguese-language (`lang="pt-PT"`) PWA-style form used to score employees of a cleaning company across three roles. The entire app — HTML markup, CSS, and ES5 JavaScript — lives in `index.html` (~1900 lines). There is no build system, no package manager, no dependencies beyond Google Fonts loaded from a CDN, and no tests.

## Running / developing

- Open `index.html` directly in a browser, or serve the directory with any static server (e.g. `python3 -m http.server`).
- There is no build, lint, or test command. Edits to `index.html` are live on reload.
- Because state lives in `localStorage`, use a private window or clear storage (`al_evaluations_v1`, `al_draft_v1`, `al_script_url`) to reset.

## Architecture

The whole app is one IIFE at the bottom of `index.html` (`(function() { 'use strict'; ... })()`). Top-level structure:

1. **`ROLES` object** (`index.html:823–1022`) — declarative data for the three roles (`operador`, `chefe`, `encarregado`). Each role has `categories[].criteria[]` (rated 1–5, lower = better), `bonus[]` items (each worth −1 to the total), `thresholds[]` mapping total points to a classification (`Excelente` … `Crítico`), plus `maxScore`. To add/change a role, edit this object — the UI is fully driven from it.
2. **Two screens**, toggled by `showScreen(name)` switching `.screen.active`:
   - `#screen-home` — role picker + history list (rendered by `renderHome` / `renderHistory`).
   - `#screen-form` — meta fields, criteria, bonus, score panel (rendered by `openForm` → `renderCategories` + `renderBonus`).
3. **Scoring model** (`calculateScore` at `index.html:1353`): sum of criterion ratings = `base`; bonus checks subtract from total; `total = base − bonus`, clamped ≥ 0. `getClassification(total)` walks `thresholds` (sorted ascending by `max`) and returns the first match.
4. **Persistence**
   - `getEvaluations` / `saveEvaluations` → `localStorage['al_evaluations_v1']` (array of saved records).
   - `saveDraft` / `loadDraft` / `clearDraft` → `localStorage['al_draft_v1']` (single in-progress form, auto-saved on every rating/bonus toggle and on every `input` event for meta fields).
   - Records have a generated `id` (`ev_<timestamp>_<rand>`); editing an existing record reuses its id (see `editingId`).
5. **Google Sheets sync** (optional) — `syncRecordToSheets` POSTs JSON as `text/plain` to a Google Apps Script Web App URL to avoid a CORS preflight. The URL is set via the gear button (`#configBtn`), validated with a `?action=ping` GET, and stored in `localStorage['al_script_url']`. The `GOOGLE_SCRIPT_URL` constant at `index.html:1033` can hard-code it instead. `updateSyncBadge` updates the header chip; sync failures fall back to local-only with a toast.
6. **Event handling** uses a single delegated `click` listener on `document` (`index.html:1751`) that walks up the DOM looking for `data-role`, `data-rating`, `data-bonus`, `data-delete`, or `data-id` attributes. Add new interactive elements by setting one of these attributes rather than attaching new listeners.

## Conventions to follow when editing

- **ES5 only.** The header comment at `index.html:816` is load-bearing: this is "ES5 puro — compatibilidade total iOS/Safari/Chrome/Edge". Use `var`, `function` declarations, `XMLHttpRequest`. **Do not** introduce `let`/`const`, arrow functions, template literals, `fetch`, `Promise`, `async/await`, optional chaining, or `for...of`. Iterate with `for (var i = 0; i < arr.length; i++)` and `for (var k in obj) { if (obj.hasOwnProperty(k)) ... }` as the existing code does.
- **All user-facing strings are Portuguese (pt-PT)** — match the existing tone (informal "tu" form, e.g. "Cola aqui o URL", "Avalia pelo menos um critério"). Toasts/modals/labels should stay in Portuguese.
- **No external libraries.** Keep everything inline in `index.html`. Don't split into separate JS/CSS files unless explicitly asked — the single-file deployability is intentional.
- **Escape any user-controlled string** before injecting into `innerHTML` — use the existing `escapeHtml` helper. Most rendering builds HTML strings, so this matters.
- **Lower rating = better.** Ratings are 1 (Excelente) … 5 (Crítico) and totals are summed *up*, so a lower total is a better evaluation. The CSS classes `cls-excelente` / `cls-bom` / `cls-suficiente` / `cls-fraco` / `cls-critico` follow this scale.
- **The `encarregado` role uses a different meta field** (`metaEquipasField` — number of teams managed) instead of `metaEquipaField` (team name). `openForm` toggles the visibility; preserve this when touching that code path.
- After any change to ratings, bonus, or meta inputs, call `saveDraft()` so the in-progress form survives reloads — this is wired through the delegated handlers, so keep new interactions going through the same data-attribute pattern.

## Git workflow

Develop on the branch `claude/add-claude-documentation-yDXFQ` and push there. Do not open a PR unless asked.
