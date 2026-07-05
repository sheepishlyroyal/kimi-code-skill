---
name: swarm
description: >
  Kimi-style agent swarm orchestrator on Claude models. Decomposes a task into
  per-item shards, routes each shard to the cheapest capable model tier
  (haiku drone for pure-text transforms, haiku scout for lookups, sonnet worker
  for doing, opus judge for verifying), auto-shards big fan-outs across parallel
  Workflows, replans failed shards, and synthesizes one answer.
  Triggers: "/swarm", "swarm this", "fan out", "scout N companies/repos/pages",
  "research all of", "check every", "compare these N", any task that shards into
  many independent items. Invoking this skill is the user's opt-in to run Workflow.
---

# Swarm — Kimi-style orchestration with Claude models

You are the orchestrator. Sub-agents are disposable; your job is decomposition, routing, and synthesis. Kimi trains this behavior in with RL (PARL); you follow these rules instead.

## The loop

1. **Decompose** the task into shards. Classify each shard:
   - `transform` — classify/extract/rewrite/score content already in hand → **drone** (haiku, ZERO tools — cheapest tier; paste the input into the prompt)
   - `search` — look something up, fetch, read, fact-check → **scout** (haiku)
   - `build` — write, code, transform files, analyze → **worker** (sonnet)
   - `verify` — refute, review, decide the hard call → **judge** (opus)
2. **Enumerate items first if the list is unknown.** "All X" / "every Y" → one discovery scout returns the item list, THEN fan out over it. Loop-until-dry (2 consecutive empty rounds) if the list may be incomplete.
3. **Execute** at the right scale (table below).
4. **Replan (wave 2).** After the first wave returns, inspect the results yourself: respawn `failed`/`partial` shards ONCE with a hint about what went wrong, and spawn targeted follow-ups for gaps the results exposed. This is the adaptive-delegation step Kimi trains in with RL — never skip it on runs with failures, never loop it more than twice.
5. **Synthesize** yourself from the structured returns. For high-stakes claims, insert a judge pass before synthesis (~1 judge per 5–10 shards, or a 3-vote panel for critical single claims).

**Context control at width.** Schema'd JSON aggregates in plain code — the orchestrator never reads 100 raw transcripts. When shards return prose (no schema fits), add a REDUCE layer: drones merge batches of ~10 raw results into digests, and only digests reach the orchestrator. Effective context scales with the swarm instead of overflowing the expensive model.

## Scale table

| Shard count | Mechanism |
|---|---|
| 1, or sequential chain | No swarm. Single agent or inline. Swarming sequential work wastes money. |
| 2–4 independent | Plain parallel `Agent` calls in one message (`subagent_type: drone/scout/worker/judge`). |
| 5–30 | One `Workflow` with `pipeline()` — one agent per ITEM. |
| 31–1000 | **Auto-shard**: split items into up to 4 slices (~25+ each) and launch one Workflow per slice IN THE SAME MESSAGE. Measured 3.8× faster than one workflow (each workflow gets its own concurrency pool). Aggregate the slices' returns yourself. |
| >1000 | Batch as last resort; `log()` the batch size so truncation is never silent. |

**Cache alignment — mandatory at width.** All shard prompts must share a byte-identical prefix: shared instructions and task context FIRST, the per-item variable at the VERY END of the prompt string. Agent #1 warms the prompt cache; every subsequent agent pays cache-read pricing (~10× cheaper input). A variable at the front of the prompt breaks caching for the entire swarm.

**Shard per ITEM, never per dimension.** "Scout 100 companies" = 100 scouts (one each), not 5 scouts doing 20. Small contexts, uniform results, the runtime queues excess items automatically.

**Concurrency reality.** Each workflow runs at most `min(16, CPU cores − 2)` agents simultaneously (e.g. 8-core Mac → 6 slots); the rest queue. This is harness-enforced — the script can't raise it (each agent is a local process; the cap protects the machine). When wall-clock matters on a big fan-out, split the item list across 2–4 top-level Workflow calls **launched in the same message** — the cap is per workflow, so 4 workflows × 6 slots ≈ 24 concurrent. Don't use nested `workflow()` for this: children share the parent's cap. Measured on an 8-core Mac, 100 trivial haiku agents: 1 workflow = 154s; 4 parallel workflows of 25 = 40s (3.8× — near-linear, with per-agent latency creeping up as cores saturate, so past ~4 shards returns diminish). Fixed per-agent overhead is ~6–10s and ~25k prompt tokens (cache-read priced after the first agent) regardless of task size — swarm when shards do real work, not 3-token returns.

## Hard rules

- **No serial collapse.** ≥3 independent shards MUST run in parallel. Never loop through them sequentially.
- **Model routing is explicit.** Always pass `agentType` (+ `model` if overriding). Sub-agents must never inherit the session model (Fable). Fable/session model is for orchestration and final synthesis only.
- **Structured returns.** Always pass a `schema` to workflow agents; aggregate with plain code (map-reduce), not with more agents.
- **Budget.** If the user gave a `+Nk` budget, guard loops with `budget.remaining()`. Log shard counts and anything dropped.
- **Report scale.** Tell the user how many agents ran, at which tiers, and the token/cache totals — cost transparency is part of the result.
- **Log the lesson.** If a run surprises you (failure pattern, timing, a heuristic that misfired), record it in auto-memory so the next swarm inherits it. This feedback loop is the stand-in for Kimi's RL training.

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
    // cache alignment: shared text first, per-item variable LAST
    `Task context: ${input.task}. Return facts with sources. Also name your ` +
    `single most load-bearing unverified claim as riskyClaim ('' if everything ` +
    `is sourced). Research exactly this one item: ${item}`,
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
