Language: English | [日本語](README.ja.md)

# agent-stocktake

[![Ask DeepWiki](https://deepwiki.com/badge.svg)](https://deepwiki.com/shimo4228/agent-stocktake)

An [Agent Skill](https://agentskills.io/specification) for Claude Code that audits your **agent definitions** (`~/.claude/agents/*.md`, the files that define subagents) and gives each one a verdict, such as Keep, Improve, Demote to skill or Dissolve. Agent definitions get their own audit because each one costs context twice: its description sits in a listing loaded into every session, and its body loads each time the agent is called (see [The hybrid cost model](#the-hybrid-cost-model)). It is the third of the author's stocktake (audit) skills, after [skill-stocktake](https://github.com/shimo4228/skill-stocktake) for skills and [rules-stocktake](https://github.com/shimo4228/rules-stocktake) for always-loaded rules.

The audit ends in a table with one row per agent, `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason`, and every non-Keep reason cites its evidence, so you can decide from the table. The skill's own example of an Improve reason:

> L23 'only report issues you are 80%+ confident in' suppresses findings per current-generation guidance — invert to 'report everything; caller filters in a separate pass'.

It edits, demotes or deletes an agent definition only after you approve that file, one at a time. The author's other work is listed under [More from the author](#more-from-the-author).

## Install

There are two routes. Cloning this repository (first block) installs agent-stocktake alone. The akc-cycle plugin (second block) also installs `skill-creator` and `adr-writer`, two skills the audit's last phase hands work to: creating a skill from an agent definition, and recording in an ADR why an agent was removed (see [Requirements](#requirements) for the clone route without them).

```bash
git clone https://github.com/shimo4228/agent-stocktake
mkdir -p ~/.claude/skills
cp -r agent-stocktake/skills/agent-stocktake ~/.claude/skills/agent-stocktake
```

With the clone install, the skill runs its evidence script at `~/.claude/skills/agent-stocktake`, so keep the folder at that path. Run it by typing `/agent-stocktake`. Claude does not start it on its own: the skill sets `disable-model-invocation: true`, so it stays out of every session's context until you call it.

The same skill also ships in the [akc-cycle](https://github.com/shimo4228/akc-cycle) Claude Code plugin, together with the other skills of the Agent Knowledge Cycle (AKC: the author's six-phase, human-gated cycle that turns a coding agent's repeated experience into skills and rules). agent-stocktake belongs to the cycle's Curate phase (the structural and semantic audit of the skills, rules and agents the cycle has accumulated), and in the plugin it is called `/akc-cycle:agent-stocktake`. This repository is synced one way from the same source, so between syncs it can trail the plugin.

```
/plugin marketplace add shimo4228/akc-cycle
/plugin install akc-cycle@akc-cycle
```

## Requirements

Only the first two are required; everything else is optional.

- Claude Code with the **Glob**, **Read**, **Edit**, **Write**, and **Bash** tools (Write updates the results ledger; the audit itself runs in your main conversation and launches no subagents).
- [`uv`](https://docs.astral.sh/uv/) and Python 3.11 or later, for the bundled evidence script.
- Optional: `jq` for the changed-mode timestamp check.
- Optional, for the clone install: skills named `skill-creator` and `adr-writer` (the akc-cycle plugin installs both). Only Phase 4 uses them (see [How It Works](#how-it-works)). Without them the audit still gives every verdict and still deletes a dissolved agent once you confirm, but you create the demoted skill and write any ADR yourself.
- Optional, from the author's harness (the author's own Claude Code configuration, the source this repository is synced from) and not shipped here: `harness_lint.py`, which checks each agent's frontmatter, that its `name` matches the filename, and that Markdown links in agent bodies resolve (the audit reads those checks from it rather than repeating them, so without it they do not run), and an agent-usage logging hook for the usage column (without its log, usage renders as `—`, unmeasured, never as 0).

## The hybrid cost model

An agent definition costs context in two ways, one like an always-loaded rule and one like a skill that loads on call:

- Its `description` is injected into **every session** via the "Available agent types" listing: **residency**, like a rule. The audit asks whether each description is dense, truthful to the body, and selection-enabling, and it tracks the *aggregate* description word count: the longer the listing, the weaker each entry's selection signal.
- Its body loads only when the agent is invoked: **invocation**, like a skill, but triggered by the model's delegation judgment rather than description matching. The audit asks whether the body is current, unique, and free of instructions that quietly degrade the current model generation (Claude 5: the skill was built during the author's Claude 4 → Claude 5 audit in July 2026, and its premises about model behaviour were set then).

## What the audit looks for in agent bodies

- **Suppression instructions**: confidence thresholds ("only report findings you are ≥N% sure of"), severity floors, "be conservative" framings. The audit's own rule is *report everything, filter in a separate pass*, because a current (Claude 5) model follows a suppression instruction literally and silently drops findings. These are **Improve-by-inversion** candidates: rewrite the instruction in the opposite direction, because deleting it leaves the suppressive frame in place.
- **Previous-generation over-constraint**: exhaustive step-by-step procedures for judgment the current (Claude 5) model holds natively, repeated emphasis, ALWAYS/NEVER pairs.
- **Absorption by Claude Code**: Claude Code itself (its built-in features and the main conversation loop) now covers the agent's job. The test is which context the role needs: roles that gain from a *fresh* context (review, adversarial verification, essence evaluation: judging whether something should be built at all) legitimately live in a subagent; roles that gain from the *rich* context of the main conversation (planning, generation) belong to the main loop by default, so for them the main loop itself counts as the absorber. SKILL.md names two exceptions that keep such a role in a subagent: the caller hands it a self-contained input packet, or its work is bulky enough to flood the main context.

## Modes

| Mode | Trigger | What it does |
|------|---------|--------------|
| **full** | default, or `/agent-stocktake full` | Read and evaluate every agent definition |
| **changed** | `/agent-stocktake changed` | Re-evaluate only files changed since the last run; carry the rest forward from the ledger (`results.json`, the file where each run records its verdicts). Reference checks still run over the full set, because a reference broken elsewhere is invisible to file mtimes |

## How It Works

1. **Phase 1 — Evidence, inventory, usage**: the bundled script (`scripts/agent_evidence.py`) measures description words and body lines, classifies each listed tool, and lists near-duplicate descriptions plus line-numbered candidates for suppression and ALWAYS/NEVER phrasing, as JSON with no verdict. The audit then checks that file paths written in agent bodies resolve and reads usage counts (from the usage logging hook, if present) as evidence with explicit lower-bound caveats.
2. **Phase 2 — Evaluation**: each agent answers eight Yes/No questions (two on the description, six on the body). Any draft verdict other than Keep is then challenged with agent-specific questions that try to refute it, and a Dissolve candidate must name its absorber concretely or the claim is refuted. The answers are evidence for the verdict, never added up into a score.
3. **Phase 3 — Summary**: an `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason` table, closing with the total description word count and its delta since the previous audit.
4. **Phase 4 — Consolidation**: candidates are confirmed **one by one**: evidence first, then `[y/n/skip]`; no bulk approval. Approved edits are applied in-session; Demote hands skill creation to a skill named `skill-creator`; Dissolve offers to record the why in an ADR through `adr-writer`.

## Verdict Criteria

| Verdict | Meaning |
|---------|---------|
| **Keep** | Earns both layers: description dense and truthful, body current and unique |
| **Improve** | Worth keeping, needs tightening; includes **inversion** of suppression instructions |
| **Update** | Referenced technology, tool, or model is outdated (verified, with evidence) |
| **Merge into [X]** | Substantial overlap with another agent |
| **Demote to skill** | The value is the instructions, not the separate context/process |
| **Dissolve** | Absorbed by Claude Code itself. Retirement by *success*: delete before the stale body overrides newer defaults; record the why in an ADR |
| **Retire** | Defect-based removal: low quality, stale, broken beyond repair |

## References

The Yes/No question design is inherited from skill-stocktake / rules-stocktake and follows checklist-based evaluation research: [BinEval "Ask, Don't Judge"](https://arxiv.org/abs/2606.27226), CheckEval (arXiv:2403.18771), TICK (arXiv:2410.03608).

## More from the author

- **[akc-cycle](https://github.com/shimo4228/akc-cycle)**: installs this skill together with the rest of the cycle as one Claude Code plugin.
- **[Agent Knowledge Cycle](https://github.com/shimo4228/agent-knowledge-cycle)**: the reasoning behind each phase of the cycle, Curate among them, recorded as dated design decisions.
- **[skill-stocktake](https://github.com/shimo4228/skill-stocktake)**: the same kind of audit for your installed skills, finding staleness, conflicts and redundancy with a verdict per skill.
- **[rules-stocktake](https://github.com/shimo4228/rules-stocktake)**: the same kind of audit for your always-loaded rules, weighing what each rule costs in every session.
- **[generation-audit](https://github.com/shimo4228/generation-audit)**: re-checks your own rules, skills and agents when a new Claude model takes over a role, and for agents proposes that you run `/agent-stocktake` with the evidence it collected.
- **[shimo4228](https://github.com/shimo4228/shimo4228)**: the author's hub, with AKC next to the other long-running practice lines and their DOIs.

## License

MIT

<details>
<summary>For tools and AI assistants</summary>

agent-stocktake is an Agent Skill for Claude Code that reads every subagent definition under `~/.claude/agents/` in one context and gives each a verdict, for people who keep their own subagents and want the listing that rides in every session kept short and the bodies kept current. The seven verdicts are Keep, Improve, Update, Merge into [X], Demote to skill, Dissolve and Retire; no agent definition is edited, demoted or deleted without a one-at-a-time `[y/n/skip]` confirmation, and without asking it writes its own files only inside its skill folder: its results ledger, plus the `.venv` that `uv` creates there on the first run of the evidence script. The one write outside that folder is uv's own cache: on that first run `uv` downloads the build backend (hatchling) and the test dependency (pytest) into it.

It exists because an agent definition pays two costs. Its description sits in the "Available agent types" listing in every session, like a rule, and the longer that listing, the weaker each entry's selection signal; its body loads only when the agent is called, like a skill, but on the model's delegation judgment. So the description is audited on residency density and truthfulness, and the body on invocation quality: suppression instructions (confidence thresholds, severity floors) that a current (Claude 5) model follows literally and that silently drop findings, previous-generation over-constraint, and absorption by Claude Code's built-in features or the main loop. The Dissolve verdict implements the Agent Knowledge Cycle's Scaffold Dissolution concept (retiring an agent once Claude Code has absorbed its job); when the model generation changes, [generation-audit](https://github.com/shimo4228/generation-audit) collects evidence and proposes that the user run `/agent-stocktake` with it, since Claude cannot start this skill on its own.

Canonical facts: MIT license; a `SKILL.md` plus one Python script, `scripts/agent_evidence.py` (Python 3.11 or later, run through `uv` with the bundled `uv.lock`, tests under `tests/`), which prints JSON evidence and never a verdict; maintained by one author (@shimo4228). Status: active, synced one way from the author's Claude Code harness by `scripts/sync-from-local.sh` (`--dry-run` reports differences only; it never commits), and also shipped in the akc-cycle plugin as `/akc-cycle:agent-stocktake`, so this repository can trail the plugin between syncs. Requirements: Claude Code with Glob, Read, Edit, Write and Bash; `uv`; `jq` for `changed` mode; no paid key. It runs only when called as `/agent-stocktake` (`disable-model-invocation: true`). Two inputs come from the author's harness and are not shipped here: the frontmatter, name/filename and Markdown-link checks are read from `harness_lint.py`, and the usage column reads `~/.claude/metrics/agent-usage.jsonl`, written by a `log-agent-usage.sh` hook (counts are lower bounds, since only Agent-tool launches are logged). The script reads the local MCP config files (`~/.claude.json`, `~/.claude/.mcp.json`) to see which servers are configured. It applies approved edits itself, hands Demote to a skill named `skill-creator`, offers `adr-writer` for Dissolve, and writes its ledger to `results.json` in the skill folder: `~/.claude/skills/agent-stocktake/results.json` for the clone install, or `${CLAUDE_PLUGIN_ROOT}/skills/agent-stocktake/results.json` for the plugin copy.

Example: `uv run --frozen --project ~/.claude/skills/agent-stocktake --directory ~/.claude/skills/agent-stocktake python scripts/agent_evidence.py --root ~/.claude` prints per-agent `desc_words`, `body_lines` and tool status, plus `total_desc_words`, `description_near_duplicates`, `suppression_candidates` and `always_never_candidates`, and exits 0 however many findings (2 only when the corpus, or a file passed with `--known-tools`, cannot be read). The audit then renders `Agent | Desc words | Body lines | Usage 90d | Verdict | Reason`. An Improve reason reads: "L23 'only report issues you are 80%+ confident in' suppresses findings per current-generation guidance — invert to 'report everything; caller filters in a separate pass'."

Links: [skills/agent-stocktake/SKILL.md](skills/agent-stocktake/SKILL.md) is the skill itself; [llms.txt](llms.txt) is the machine-readable summary. The skill extends the Curate phase of the [Agent Knowledge Cycle (AKC)](https://github.com/shimo4228/agent-knowledge-cycle) to the agent-definition layer, concept DOI [10.5281/zenodo.19200726](https://doi.org/10.5281/zenodo.19200726); cite AKC by that DOI. The installable form of the whole cycle is [akc-cycle](https://github.com/shimo4228/akc-cycle).

</details>
