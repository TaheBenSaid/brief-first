# brief-first

Make your AI coding agent write a plan before it writes code, then build from that plan.

When a model plans and types at the same time, it guesses, and you find the wrong guesses in the diff. With a brief you read the plan in two minutes and fix it before a single line exists. The builder then gets exact files and line numbers instead of "add some chips somewhere".

It works with one model doing both phases, or with a different model for each. The brief matters more than who writes it.

## What a brief has

| Section | What goes in it |
| --- | --- |
| Goal | One or two sentences |
| Context | What was found in the code: files, line numbers, things to reuse, traps |
| Steps | Numbered, exact files, in order |
| Rules | The project's constraints for this task |
| Verify | Commands that prove it works |
| Open questions | Real blockers only, or "None" |

The blank template is [`TEMPLATE.md`](TEMPLATE.md). A real brief for a small Flutter feature (filter chips, English and Arabic) is in [`example`](example/activity-filter-chips.md). That one caught a token with a misleading name and asked for chips that wrap, so they don't run off the screen with large system fonts.

## Use it as a skill

Clone it into your agent's skills folder, for example:

```bash
git clone https://github.com/TaheBenSaid/brief-first.git ~/.claude/skills/brief-first
```

It works with Claude Code or any agent that supports the Agent Skills format. Then ask something like "plan first: add a dark mode toggle to settings". The agent reads the code, writes the brief to `.claude/plans/`, and stops. You read it, change what's off, say go, and it builds from the file and runs the checks in Verify.

## Use it without an agent

Copy `TEMPLATE.md` into your project, fill it in yourself or paste it into any chat with "fill this in for my task, don't write code yet". Then hand the filled brief over as the instructions.

## Plan with one model, build with another

This is the setup I tested in Claude Code: one model writes the brief, another builds from it. The switch happens on its own.

| Phase | Model | Where it's set |
| --- | --- | --- |
| Plan | Fable 5.2 | `model: fable` in `agents/planner.md` |
| Build | Opus 5.5 | `"model": "claude-opus-5-5"` in `~/.claude/settings.json` |

1. Copy the planner agent:

   ```bash
   cp brief-first/agents/planner.md ~/.claude/agents/planner.md
   ```

2. Set the main model, the one that builds, in `~/.claude/settings.json`:

   ```json
   { "model": "claude-opus-5-5" }
   ```

3. Add this rule to `~/.claude/CLAUDE.md` so every session hands planning to the planner first:

   ```markdown
   ## Plan first
   For any task that edits several files or takes more than a few steps,
   first run the `planner` subagent with the full task. It writes a brief
   to `.claude/plans/` and returns the path. Read that file, raise any open
   question that blocks the work, then build it step by step and run every
   check under Verify. Skip the planner for questions and one-line fixes.
   ```

Swap the model names for whatever you trust most for thinking and for coding. The brief format stays the same.

Honest numbers from my one test (a small Flutter feature): the two-model run took about 2 minutes 20 seconds against 33 seconds for one model doing it all, and used more tokens. The plan is what made the result better, so I only use the split on bigger or riskier tasks.

## When not to

One-line fixes, renames, or anything you already know exactly how to change. Just ask for those directly.

## More tools

Part of a series of small tools for Flutter developers, each tied to something I write about. The full list is in [flutter-skills](https://github.com/TaheBenSaid/flutter-skills).

## License

MIT
