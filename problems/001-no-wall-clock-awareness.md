# 001 — Long-running sessions are time-blind across turns

**Status:** fix in progress
**Surfaces in:** Claude Code, Claude Agent SDK (any wrapper around the same closed binary)
**Fix repo:** [claude-agent-clock](https://github.com/zynskibud/claude-agent-clock)

---

## Observation

A Claude Code session that has been running for ~12+ hours will confidently reason about "today" using the date that was anchored in the system prompt at session start — even when the local date has rolled hours ago.

Concretely: start a session at 11pm, leave it idle, come back at 2am the next day, ask a date-sensitive question. The model answers as if it's still the previous day. Same problem for "the deadline is in 4 hours" type prompts — the deadline can pass mid-session and the model has no way to notice.

The only timestamps the model ever sees are whatever was injected at session start. There is no built-in re-stamping between turns.

## Why this happens (best guess)

The Claude Code binary constructs a system prompt at session start that includes the current date, then doesn't refresh it. The agent loop treats the session as one long conversation; nothing in the loop says "before each user turn, re-stamp the wall clock."

Both `claude-agent-sdk-typescript` and `claude-agent-sdk-python` are wrapper packages around the same closed binary (the TS SDK lists 8 platform-specific binary `optionalDependencies`; the Python SDK README confirms the binary is bundled). So the gap exists upstream of either SDK — neither wrapper repo is the right place to fix it.

The model is not the problem here. Given a turn that includes the current ISO timestamp, it reasons about time correctly. It's a context-construction problem.

## What the fix looks like

**Native fix (right place):** in the agent loop inside the Claude Code binary, on each new user turn, re-stamp the system context with the current ISO timestamp and local date. Probably a one-line change for whoever owns the loop.

**Default-on hook (also fine):** ship a `UserPromptSubmit` hook in Claude Code's default settings that does the same thing in user-space.

**External wrapper (the bandaid I built):** [claude-agent-clock](https://github.com/zynskibud/claude-agent-clock) implements this as a `UserPromptSubmit` hook you can drop into `~/.claude/settings.json`, plus a `HookCallback` factory for direct Agent SDK consumers. Fires when (a) gap since last turn exceeds a threshold, (b) local date has rolled, or (c) a configured deadline has passed. Event-based, not interval-based — interval ticks would become noise the model learns to filter.

## Why this matters

Without time awareness, the model can't:

- Reliably reason about deadlines that pass mid-session
- Notice that "the user walked away for 3 hours" is significant
- Distinguish "I just did this" from "I did this yesterday and the cache is stale"
- Self-pace any kind of long-running ambient task ("check on the deploy every 30 min")

This is foundational for any agent that's supposed to operate over hours-to-days timelines, not just one-shot prompts.

## Open questions

- Does Anthropic intend Claude Code to re-anchor the wall clock natively at some point? (No public statement that I've found.)
- Is there a less obvious reason re-anchoring isn't done already — e.g. would it invalidate prompt caching in a way that matters?
- What's the right `minGapMinutes` default? My current guess is 30, but I haven't tested whether 15 or 60 produces meaningfully different model behavior.
