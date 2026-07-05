# claude-swarm

Kimi-style agent swarm for [Claude Code](https://claude.com/claude-code) — with Claude models.

Moonshot's Kimi K2.5/K2.6 ships "Agent Swarm": the model decomposes a task, spawns up to 300 sub-agents in parallel, and synthesizes the results. That orchestration behavior is trained into Kimi's weights with RL (PARL). This repo recreates the same behavior in Claude Code as a **policy layer** — a skill, three tiered agents, and a standing dispatch rule — with one upgrade: every shard runs on the cheapest *Claude* model that can handle it.

```
task ──▶ orchestrator (your session model)
           │  decompose → route → fan out
           ├─▶ scout  × N   (Haiku)   lookups, fetches, fact-checks — one per ITEM
           ├─▶ worker × N   (Sonnet)  implementation / analysis shards
           ├─▶ judge  × few (Opus)    adversarial verification, ~1 per 5–10 shards
           │
           └─◀ structured JSON returns → map-reduce → one synthesized answer
```

"Scout 100 companies" = 100 Haiku scouts, one per company. The wide layer (most of the token volume) runs ~15× cheaper than Opus-class models; the expensive models only touch the small, hard slices.

## What's in the box

| Path | What it is |
|---|---|
| `skills/swarm/SKILL.md` | The `/swarm` orchestrator: decompose → route → execute → synthesize loop, scale table, anti-patterns (no serial collapse, no swarm-for-swarm's-sake), and a ready-to-adapt Workflow script template |
| `agents/scout.md` | Haiku, read-only. One item per scout, compact sourced findings, 2–5 tool calls |
| `agents/worker.md` | Sonnet, full tools. Owns one shard, verifies before reporting |
| `agents/judge.md` | Opus. Default-to-refute verifier; concrete evidence or it doesn't count |
| `CLAUDE.md.snippet` | Standing dispatch policy: when to swarm autonomously, routing table, scale rules |

## Install

```bash
git clone https://github.com/sheepishlyroyal/claude-swarm
cd claude-swarm
mkdir -p ~/.claude/skills/swarm ~/.claude/agents
cp skills/swarm/SKILL.md ~/.claude/skills/swarm/
cp agents/*.md ~/.claude/agents/
cat CLAUDE.md.snippet >> ~/.claude/CLAUDE.md   # optional: autonomous dispatch
```

Agents register on the next Claude Code session start. The skill is picked up immediately.

The `CLAUDE.md.snippet` step is what makes swarming *autonomous* — it's a standing authorization telling Claude to fan out whenever a task shards into ≥3 independent units, without being asked. Skip it if you'd rather trigger swarms explicitly with `/swarm`.

## Use

```
/swarm scout these 40 companies for pricing, funding, and headcount: <list>
/swarm review every file in src/ for error-handling gaps
/swarm compare these 12 libraries on bundle size, license, maintenance
```

Or, with the CLAUDE.md snippet installed, just ask normally — wide tasks fan out on their own; deep/sequential tasks stay single-agent.

## Design notes

- **Shard per item, not per dimension.** One agent per company/file/page keeps contexts small and returns uniform. The Workflow runtime queues past its ~16-concurrent cap automatically (up to 4096 items per pipeline).
- **Structured returns, code-side reduce.** Every shard returns schema-validated JSON; aggregation is plain JavaScript in the workflow script, not another LLM call.
- **Kimi's RL lessons as rules.** PARL exists to prevent "serial collapse" (the orchestrator lazily running one agent at a time) and reward useful delegation. Here those are hard rules in the skill: ≥3 independent shards must run in parallel; sequential chains must not be swarmed.
- **Judges are rationed.** Opus only sees load-bearing unverified claims (each scout nominates its riskiest one), not every shard.

## Verified

The template was exercised live before publishing: a 3-item fan-out ran 3 parallel Haiku scouts, each returning schema-validated findings with source URLs, 0 failures, ~39s wall clock, ~85k sub-agent tokens (cents).

## Requirements

Claude Code with the `Agent` and `Workflow` tools (any recent version). No dependencies.

## License

MIT
