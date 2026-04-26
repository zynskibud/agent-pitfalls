# agent-pitfalls

A running catalog of problems I keep hitting while using LLM agents (Claude Code, Claude Agent SDK, Codex, Cursor, etc.).

Each entry is a focused write-up: **what I observed → my best guess at the root cause → what a fix looks like → status**.

When I ship a fix, the entry links out to a standalone repo. When I haven't, the entry stands as a reproducible bug report someone else can pick up.

---

## Philosophy

Most of these are not "the model is dumb." They're **harness problems**: the gap between what the model is capable of in principle and what the surrounding agent loop actually exposes to it.

The harness is the engineering surface where 80% of the wins live, and right now it's where 80% of the friction is too. Most of the entries here will be observations about the seam between *model capability* and *tool design*.

The ones that look like one-liners (e.g. inject the wall clock between turns) usually are. The ones that look like architecture problems (e.g. web search returns thin snippets and the agent has no way to deepen) require more work.

---

## Index

| # | Problem | Status | Fix |
|---|---|---|---|
| [001](./problems/001-no-wall-clock-awareness.md) | Long-running sessions are time-blind across turns | **fix in progress** | [claude-agent-clock](https://github.com/zynskibud/claude-agent-clock) |
| [002](./problems/002-web-search-too-shallow.md) | Built-in WebSearch misses information that surfaces immediately via real search | open | — |

---

## Adding a new entry

```
problems/NNN-short-slug.md
```

Use [`problems/_template.md`](./problems/_template.md). Bump the index in this README. One file per problem, numbered sequentially. If a fix ships, link it from both the entry and the index.

---

## License

MIT — see [LICENSE](./LICENSE).
