# multi-agent-review-cc

A Claude Code skill that panels six independent review agents across three topics before you execute a plan or finish a spec. When model tiers disagree, an opus juror adjudicates. The result is a gated **Blockers / Warnings / Observations** verdict.

## What it does

```
/multi-agent-review spec    # fires after brainstorming, before writing-plans
/multi-agent-review plan    # fires after writing-plans, before subagent-driven-development
```

Six agents run in parallel — haiku and sonnet tiers each review **completeness**, **alignment**, and **risk**. If the two tiers disagree on a finding, a single opus juror adjudicates. If they agree, no juror is invoked.

The verdict gates the next step:
- **Blockers** → stops execution, presents to operator (fix + re-run, or override with logged note)
- **Warnings** → presents to operator (fix or accept and continue)
- **Clean** → auto-proceeds to next skill

## Install

Copy the skill files into `~/.claude/skills/multi-agent-review/`:

```bash
git clone https://github.com/riekelt/multi-agent-review-cc ~/.claude/skills/multi-agent-review
```

The skill is immediately available as `/multi-agent-review` in any Claude Code session.

## Project-specific rules

Copy `project-rules.example.md` to your project root as `project-rules.md` and customise it. The skill injects it into the alignment and risk reviewers. Without it, only generic checks run.

## Flags

| Flag | Effect |
|---|---|
| `--fast` | Haiku-only, skip sonnet tier, skip juror. ⚠ Do not use on plans that touch production-safety paths. |

## Panel failure semantics

The skill is designed to fail closed:

| Failure | Behaviour |
|---|---|
| < 3 valid agent reports | Halt — no verdict produced |
| 3–5 valid reports | Continue with panel-health WARNING |
| Juror error/timeout | Conservative fallback — all contested findings → BLOCKER |
| All agents fail | Halt — empty panel never triggers clean verdict |

## Files

| File | Purpose |
|---|---|
| `SKILL.md` | Coordinator: locates artifact, dispatches agents, compares, invokes juror, decision gate |
| `completeness-reviewer.md` | Agent prompt: TBDs, missing criteria, undefined references |
| `alignment-reviewer.md` | Agent prompt: mockup consistency, standing rules, type contracts |
| `risk-reviewer.md` | Agent prompt: production safety, fail-closed paths, blast radius |
| `synthesis-agent.md` | Opus juror prompt: rules on contested findings only |
| `project-rules.example.md` | Template for project-specific standing rules |
