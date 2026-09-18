# CLAUDE.md — Meme Machine project instructions

Project-level instructions for working in this repo. `plan.md` is the
source of truth for scope, phase order, and technical decisions — read it
before making changes. This file governs *how* to work, not *what* to
build.

## Workflow

- Before making any nontrivial change (a new phase, a design change, a
  scope change), enter plan mode: present the plan, wait for explicit
  approval, then execute. Don't skip straight to editing files on
  ambiguous or multi-step asks.
- Build in the phase order defined in `plan.md`. Check off each item as
  it's completed and tested — don't mark something done until it's been
  verified against that phase's exit criteria.
- Test each phase before moving to the next one.

## Git / GitHub conventions

- **Every commit and push to GitHub must leave no trace of AI
  assistance.** Specifically:
  - No `Co-Authored-By: Claude` (or any AI) trailer in commit messages.
  - No mention of Claude, AI, or "assistant" anywhere in commit messages,
    PR titles/descriptions, or code comments.
  - Commit messages read as if written by the human developer alone —
    plain, first-person-implied, no attribution footer of any kind.
- Standard commit hygiene otherwise: concise subject line in imperative
  mood ("add hand-tracking counter", not "added" or "adds"), body only
  when the "why" isn't obvious from the diff.
- Never force-push, never skip hooks, never rewrite already-pushed history
  without being explicitly asked.
- Confirm the target remote/repo before the first push of a session if it
  hasn't been set up yet.

## Project structure

- `index.html` — the single-page app; all three features live here (no
  build step, no framework).
- `assets/` — meme reference images used by the Mood Meme feature.
- `plan.md` — phased build plan; check here for current scope, feature
  mapping, and open decisions before assuming.
- `CLAUDE.md` — this file.