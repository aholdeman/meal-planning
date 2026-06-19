# Meal Planning

A flat-file meal planning system, driven entirely through conversation with
Claude (Claude Code, claude.ai, or the Claude mobile app) -- no app or script
to run. See [CLAUDE.md](CLAUDE.md) for the full schema and the algorithms
Claude follows for planning weeks, saving recipes, logging feedback, and
managing the freezer/pantry.

## Clone

```
git clone https://github.com/aholdeman/meal-planning.git
```

## Day to day
- Open this folder in Claude Code (or point claude.ai at this repo) and ask
  things like "plan 3 meals this week, one from the freezer and one
  mexican," or paste in a recipe link/text to save it.
- Claude pulls before reading state and pushes after every change, so both
  copies of this repo should generally stay in sync automatically during a
  chat session.
- You can also edit/commit/push by hand at any time with plain git.
