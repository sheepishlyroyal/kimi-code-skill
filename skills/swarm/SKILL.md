---
name: swarm
description: >
  Kimi-style agent swarm orchestrator on Claude models. Decomposes a task into
  per-item shards, routes each shard to the cheapest capable model tier
  (haiku scout for lookups, sonnet worker for doing, opus judge for verifying),
  runs them massively in parallel via Workflow, and synthesizes one answer.
  Triggers: "/swarm", "swarm this", "fan out", "scout N companies/repos/pages",
  "research all of", "check every", "compare these N", any task that shards into
  many independent items. Invoking this skill is the user's opt-in to run Workflow.
---

# Swarm — Kimi-style orchestration with Claude models

You are the orchestrator. Sub-agents are disposable; your job is decomposition, routing, and synthesis. Kimi trains this behavior in with RL (PARL); you follow these rules instead.

## The loop

1. **Decompose** the task into shards. Classify each shard:
   - `search` — look something up, fetch, read, fact-check → **scout** (haiku)
   - `build` — write, code, transform, analyze → **worker** (sonnet)
   - `verify` — refute, review, decide the hard call → **judge** (opus)
2. **Enumerate items first if the list is unknown.** "All X" / "every Y" → one discovery scout returns the item list, THEN fan out over it. Loop-until-dry (2 consecutive empty rounds) if the list may be incomplete.
3. **Execute** at the right scale (table below).
4. **Synthesize** yourself from the structured returns. For high-stakes claims, insert a judge pass before synthesis (~1 judge per 5–10 shards, or a 3-vote panel for critical single claims).

## Scale table

| Shard count | Mechanism |
|---|---|
| 1, or sequential chain | No swarm. Single agent or inline. Swarming sequential work wastes money. |
| 2–4 independent | Plain parallel `Agent` calls in one message (`subagent_type: scout/worker/judge`). |
| 5–1000 | `Workflow` with `pipeline()` — one agent per ITEM. |
| >1000 | Batch as last resort; `log()` the batch size so truncation is never silent. |

**Shard per ITEM, never per dimension.** "Scout 100 companies" = 100 scouts (one each), not 5 scouts doing 20. Small contexts, uniform results, the runtime queues past the ~16-concurrent cap automatically.

## Hard rules

- **No serial collapse.** ≥3 independent shards MUST run in parallel. Never loop through them sequentially.
- **Model routing is explicit.** Always pass `agentType` (+ `model` if overriding). Sub-agents must never inherit the session model (Fable). Fable/session model is for orchestration and final synthesis only.
- **Structured returns.** Always pass a `schema` to workflow agents; aggregate with plain code (map-reduce), not with more agents.
- **Budget.** If the user gave a `+Nk` budget, guard loops with `budget.remaining()`. Log shard counts and anything dropped.
- **Report scale.** Tell the user how many agents ran and at which tiers — cost transparency is part of the result.

## Workflow template

Adapt, don't rebuild. `ITEMS` comes from the request or a discovery scout.

```js
export const meta = {
  name: 'swarm-run',
  description: 'Per-item fan-out: scout each item, judge the risky ones',
  phases: [{ title: 'Scout' }, { title: 'Judge' }],
}
const SCOUT_SCHEMA = {
  type: 'object',
  properties: {
    item: { type: 'string' },
    status: { enum: ['ok', 'partial', 'failed'] },
    findings: { type: 'array', items: { type: 'object', properties: {
      fact: { type: 'string' }, source: { type: 'string' } } } },
    riskyClaim: { type: 'string' },   // most load-bearing unverified claim, '' if none
  },
  required: ['item', 'status', 'findings'],
}
const VERDICT_SCHEMA = {
  type: 'object',
  properties: { verdict: { enum: ['confirmed', 'refuted', 'uncertain'] },
    evidence: { type: 'string' } },
  required: ['verdict', 'evidence'],
}

// normalize args defensively — they may arrive stringified or absent
const input = typeof args === 'string' ? JSON.parse(args) : (args || {})
const items = input.items || []
if (!items.length) throw new Error('no items passed via args.items')

phase('Scout')
log(`Fanning out ${items.length} scouts (haiku), one per item`)
const results = await pipeline(
  items,
  item => agent(
    `Research exactly this one item: ${item}. Task context: ${input.task}. ` +
    `Return facts with sources. Also name your single most load-bearing ` +
    `unverified claim as riskyClaim ('' if everything is sourced).`,
    { agentType: 'scout', label: `scout:${item}`, phase: 'Scout', schema: SCOUT_SCHEMA }
    // if 'scout' isn't registered (fresh install, same-session), drop agentType and
    // pass { model: 'haiku', effort: 'low' } instead — identical routing
  ),
  // judge only shards that carry a risky claim — opus is expensive
  (r, item) => (r && r.riskyClaim)
    ? agent(
        `Adversarially verify, default to refute: "${r.riskyClaim}" (re: ${item}).`,
        { agentType: 'judge', label: `judge:${item}`, phase: 'Judge', schema: VERDICT_SCHEMA }
      ).then(v => ({ ...r, verdict: v }))
    : r
)
const ok = results.filter(Boolean)
log(`${ok.length}/${items.length} shards returned; ` +
    `${ok.filter(r => r.verdict).length} judged`)
return { results: ok, failed: items.length - ok.length }
```

Swap the scout stage for `agentType: 'worker'` (+ `isolation: 'worktree'` if shards edit files in parallel) when shards are `build` work.

## When NOT to swarm

- Step N needs step N−1's output (debugging chains, migrations with ordering) → single worker.
- The whole answer is one lookup → one scout, or just answer.
- Conversation, opinion, small edits → no agents at all.
