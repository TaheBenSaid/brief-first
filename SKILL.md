---
name: brief-first
description: Plan a coding task as a written brief before any code changes, then build from it. Use for features that touch several files, unfamiliar parts of a codebase, risky areas (auth, payments, security, data migrations), or when the user says "plan first", "write a brief", "don't code yet", or wants to review the approach before the code.
---

# Brief first

Two phases. Don't mix them.

## Phase 1: write the brief

1. Read before deciding. Open the files the task touches and the ones around them: how similar features are built, which widgets and helpers already exist, which tokens and strings are there, the project's rules (README, CLAUDE.md, lint config).
2. Don't edit any project file in this phase.
3. Write the brief to `.claude/plans/<yyyy-mm-dd>-<short-task-name>.md`, following `TEMPLATE.md` in this skill's folder. Six sections: Goal, Context, Steps, Rules, Verify, Open questions.
4. Write it for someone who can't read your mind:
   - exact file paths and line numbers, not "the settings screen"
   - names of things to reuse, with signatures
   - anything that looks right but is wrong (a misleading token name, a generated file, a widget that doesn't follow `Directionality`)
   - edge cases: right-to-left languages, large system fonts, empty and error states, offline
5. Stop. Show the user the file path and a 3 to 5 line summary: the approach, the files that change, any open question. Wait for a go-ahead or edits.

## Phase 2: build from the brief

1. Re-read the brief file, including any edits the user made.
2. Follow the steps in order. If a step turns out wrong once you're in the code, stop and say so instead of improvising a different design.
3. Run every check in Verify and report each result.
4. Mention anything you did differently from the brief, and why.

## When to skip it

One-line fixes, renames, copy changes, or anything where the user already said exactly what to change. Just do those.

An example brief from a real Flutter task is in `example/activity-filter-chips.md`.
