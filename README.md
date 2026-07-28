Language: English | [日本語](README.ja.md)

# agent-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/agent-stocktake) [![GitMCP](https://img.shields.io/endpoint?url=https://gitmcp.io/badge/shimo4228/agent-stocktake)](https://gitmcp.io/shimo4228/agent-stocktake)

An [Agent Skill](https://agentskills.io/specification) that audits your **agent definitions** (`~/.claude/agents/*.md`) for quality. It is the third stocktake sibling, next to [skill-stocktake](https://github.com/shimo4228/skill-stocktake) and [rules-stocktake](https://github.com/shimo4228/rules-stocktake), and it exists because an agent definition pays **both** of their cost models at once.

## The hybrid cost model

A skill's cost is trigger pollution; a rule's cost is residency. An agent has both:

- Its `description` is injected into **every session** via the "Available agent types" listing — **residency**, like a rule. The audit asks whether each description is dense, truthful to the body, and selection-enabling, and it tracks the *aggregate* description word count: the longer the listing, the weaker each entry's selection signal.
- Its body loads only when the agent is invoked — **invocation**, like a skill, but triggered by the model's delegation judgment rather than description matching. The audit asks whether the body is current, unique, and free of instructions that quietly degrade the current model generation.

## What the body screen catches

- **Suppression instructions** — confidence thresholds ("only report findings you are ≥N% sure of"), severity floors, "be conservative" framings. Current-generation guidance is *report everything, filter in a separate pass*; a suppression instruction is followed literally and silently drops findings. These are **Improve-by-inversion** candidates: rewrite the instruction in the opposite direction — deleting it leaves the suppressive frame in place.
- **Previous-generation over-constraint** — exhaustive step-by-step procedures for judgment the current model holds natively, repeated emphasis, ALWAYS/NEVER pairs.
- **Substrate absorption** — the harness now covers the agent's job natively. Judged with a fresh/rich context axis: roles that gain from *fresh* context (review, adversarial verification, essence evaluation) legitimately live in a subagent; roles that gain from *rich* context (planning, generation) belong to the main loop, so for them the main loop itself counts as an absorber.

## Modes

| Mode | Trigger | What it does |
|------|---------|--------------|
| **full** | default, or `/agent-stocktake full` | Read and evaluate every agent definition |
| **changed** | `/agent-stocktake changed` | Re-evaluate only files changed since the last run; carry the rest forward from the ledger. Mechanical integrity checks still run over the full set — cross-reference breakage is invisible to file mtimes |

## How It Works

1. **Phase 1 — Inventory + mechanical integrity checks**: enumerate `~/.claude/agents/*.md`, measure description words and body lines live, verify frontmatter, name/filename agreement, tool existence, and reference resolution. Usage counts (from an invocation-logging hook, if present) enter as evidence with explicit lower-bound caveats — unmeasured is rendered as `—`, never as 0.
2. **Phase 2 — Evaluation**: a two-stage binary screen. Stage 1 is a seven-question Yes/No checklist per agent (two questions for the description layer, five for the body layer). Stage 2 generates agent-specific refutation questions that pressure-test any non-Keep draft verdict; a Dissolve candidate must name its absorber concretely or the claim is refuted. Binary answers are evidence for a holistic verdict, never aggregated into a score.
3. **Phase 3 — Summary**: an `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` table, closing with the total description word count and its delta since the previous audit.
4. **Phase 4 — Consolidation**: candidates are confirmed **one by one** — evidence first, then `[y/n/skip]`; no bulk approval. Approved edits are applied in-session; Demote hands skill creation to a skill-creator; Dissolve offers to record the why in an ADR.

## Verdict Criteria

| Verdict | Meaning |
|---------|---------|
| **Keep** | Earns both layers: description dense and truthful, body current and unique |
| **Improve** | Worth keeping, needs tightening — includes **inversion** of suppression instructions |
| **Update** | Referenced technology, tool, or model is outdated (verified, with evidence) |
| **Merge into [X]** | Substantial overlap with another agent |
| **Demote to skill** | The value is the instructions, not the separate context/process |
| **Dissolve** | Absorbed by the substrate. Retirement by *success* — delete before the stale body overrides newer defaults; record the why in an ADR |
| **Retire** | Defect-based removal: low quality, stale, broken beyond repair |

## Install

### Claude Code

```bash
cp -r skills/agent-stocktake ~/.claude/skills/agent-stocktake
```

### SkillsMP

```bash
/skills add shimo4228/agent-stocktake
```

## Requirements

- Claude Code with the **Glob**, **Read**, **Edit**, and **Bash** tools (the audit runs in one main context — no subagents required).
- Optional: `jq` for the changed-mode timestamp check, and an agent-usage logging hook for the usage column. The skill degrades gracefully without both.

## Syncing from the harness

The canonical copy of this skill lives in the author's live Claude Code harness. This repository is a one-way publication mirror:

```bash
scripts/sync-from-local.sh --dry-run   # report differences only
scripts/sync-from-local.sh             # apply to working tree (never commits)
```

## References

The two-stage binary-question design (screen → verdict pressure-test, holistic verdict, no score aggregation) is inherited from skill-stocktake / rules-stocktake and follows the checklist-decomposition evaluation line: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226), CheckEval (arXiv:2403.18771), TICK (arXiv:2410.03608) — over-decomposition degrades correlation on holistic quality, hence seven questions and no score. The absorption question and Dissolve verdict implement the **Scaffold Dissolution** concept of the Agent Knowledge Cycle, including its model-generation trigger — on a generation change, [generation-audit](https://github.com/shimo4228/generation-audit) collects runtime-layer evidence and hands the agents slice to this skill.

## About this skill

This skill extends the **Curate** phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) — a Zenodo-citable six-phase bidirectional growth loop ([DOI 10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726)) — to the agent-definition layer, completing the stocktake set: [skill-health](https://github.com/shimo4228/skill-health) covers structural debt, [skill-stocktake](https://github.com/shimo4228/skill-stocktake) covers skills, [rules-stocktake](https://github.com/shimo4228/rules-stocktake) covers always-loaded rules, and agent-stocktake covers the definitions whose descriptions ride in every session. AKC is one of three research lines by [@shimo4228](https://github.com/shimo4228), alongside [Contemplative Agent](https://github.com/shimo4228/contemplative-agent) ([DOI 10.5281/zenodo.19212118](https://doi.org/10.5281/zenodo.19212118)) and [Agent Attribution Practice (AAP)](https://github.com/shimo4228/agent-attribution-practice) ([DOI 10.5281/zenodo.19652013](https://doi.org/10.5281/zenodo.19652013)).

## License

MIT
