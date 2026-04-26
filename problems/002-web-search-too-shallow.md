# 002 — Built-in WebSearch misses information that surfaces immediately via real search

**Status:** open
**Surfaces in:** Claude Code (likely Agent SDK too)
**Fix repo:** —

---

## Observation

The built-in `WebSearch` tool in Claude Code routinely fails to find information that:

- Shows up on the first page of a manual Google search
- Surfaces immediately via Perplexity / Exa / Tavily / DuckDuckGo
- Is found by `deep-research-pro` (a multi-source skill that uses DuckDuckGo + synthesis) on the same query

Failure modes I've watched:

1. **Returns 3-5 thin snippets** and then the agent acts as if "no good results exist" rather than deepening
2. **Doesn't follow links** when a snippet is clearly the right entry point — no automatic `WebFetch` of the top result
3. **Ignores recency** even when the query is about something that broke in the last week — gets cached pre-event content
4. **Doesn't reformulate** failed queries — same wording, same bad results, same "I don't have current information" response
5. **Doesn't compose with other tools** — knows `WebFetch` exists but doesn't pipeline `WebSearch` → `WebFetch` → `WebSearch` against extracted entities

The pattern: the agent's *single-shot* search is shallow, and the harness doesn't encourage iterative search the way a human researcher (or a dedicated research skill) does.

## Why this happens (best guess)

The `WebSearch` tool in Claude Code appears to be a single API call returning abridged snippets. The model doesn't get to:

- See the full ranked result list (top 20+, not top 5)
- Inspect raw HTML or full-text snippets to judge relevance
- Programmatically rerank or filter
- Trigger follow-up scrapes on the most promising hits without explicit prompting

So the model is forced to make a relevance call from a thin view, gets the wrong call, and lacks the affordance to course-correct without the user noticing and prompting.

Compare to `deep-research-pro` (a community skill): it explicitly plans 3-5 sub-queries, runs each, scrapes results, synthesizes. That structure is doing 90% of the work — it's not that DDG is better than whatever Anthropic is calling, it's that the *loop around the search* is more aggressive.

This is mostly a **harness design** problem, not a search backend problem. Even with a perfect search backend, a one-shot `WebSearch(query) → 5 snippets → done` pattern would still miss information that requires iteration.

## What the fix looks like

Several plausible directions, low to high lift:

**Low lift — better defaults:**
- Return more results by default (top 20 instead of top 5) and let the model triage
- Auto-invoke `WebFetch` on the top 1-3 results and include extracted text in the same tool response, not a separate turn
- Add a `recency_bias` parameter that's on by default for queries containing temporal markers ("latest", "today", "this week", etc.)

**Medium lift — iterative search primitive:**
- A new tool `DeepSearch(query)` that runs N sub-queries internally, scrapes top hits, and returns a synthesized answer + citation list
- Effectively bake the `deep-research-pro` skill pattern into the harness as a first-class tool

**Higher lift — search agent loop:**
- Spawn a sub-agent specifically for research with its own tighter loop (search → fetch → re-search), report back summarized findings
- Anthropic likely already has internal versions of this — the question is exposing it as a default tool, not a skill the user has to install

**External wrapper / user-space:**
- Could write a Claude Code skill that wraps `WebSearch` + `WebFetch` in an iterative loop and returns better-synthesized output
- Would not fix the underlying issue (most users wouldn't install it), but useful as a reference implementation

## Why this matters

Web search is the single most common tool call in any non-trivial agent session. If it underperforms a manual Google search by a meaningful margin, the agent feels dumb for reasons that have nothing to do with the model's intelligence.

The downstream effect: users learn not to trust the agent for any current-events / recently-changed / niche-domain question, and bypass it. That's a much bigger trust hit than the underlying tool failure deserves, because the user attributes the failure to the model rather than the harness.

## Open questions

- What backend does `WebSearch` actually call? (Bing? Brave? An internal Anthropic search?) The behavior characteristics matter a lot for what the fix shape is.
- How configurable is the result count from the model side? Is the 5-snippet cap a hard limit or a soft default the model could request more from?
- Does Anthropic already have a "research mode" internally that's not yet shipped to the consumer harness?
- Is the right unit of fix "a better `WebSearch` tool" or "a new `Research` tool that subsumes WebSearch + WebFetch"?
- Comparable benchmark: what does the success rate look like on a fixed set of 50 queries across `WebSearch`, `deep-research-pro`, Perplexity API, manual Google? Hasn't been measured rigorously yet.
