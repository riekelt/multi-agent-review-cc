# multi-agent-review-cc

A Claude Code / Codex skill that panels six independent review agents across three topics before you execute a plan or finish a spec. When model tiers disagree, an opus juror adjudicates. The result is a gated **Blockers / Warnings / Observations** verdict.

## Install

### Via plugin manager (Claude Code / Codex)

Add the plugin in your agent settings:
```
https://github.com/riekelt/multi-agent-review-cc
```

### Manual (Claude Code)

```bash
git clone https://github.com/riekelt/multi-agent-review-cc \
  ~/.claude/plugins/multi-agent-review-cc
```

The skill is immediately available as `/multi-agent-review`.

## Usage

```
/multi-agent-review spec    # fires after brainstorming, before writing-plans
/multi-agent-review plan    # fires after writing-plans, before subagent-driven-development
```

Six agents run in parallel — haiku and sonnet tiers each review **completeness**, **alignment**, and **risk**. If the two tiers disagree on a finding, a single opus juror adjudicates. If they agree, no juror is invoked.

The verdict gates the next step:
- **Blockers** → stops execution, presents to operator (fix + re-run, or override with logged note)
- **Warnings** → presents to operator (fix or accept and continue)
- **Clean** → auto-proceeds to next skill

## Project-specific rules

Copy `skills/multi-agent-review/project-rules.example.md` to your project root as `project-rules.md` and customise it. The skill injects it into the alignment and risk reviewers as project context. Without it, only generic checks run.

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

## Repository layout

```
.claude-plugin/
  plugin.json          Claude Code marketplace metadata
  marketplace.json     Marketplace listing

.codex-plugin/
  plugin.json          Codex marketplace metadata (includes full interface block)

skills/
  multi-agent-review/
    SKILL.md                     Coordinator: locates artifact, dispatches agents,
                                 compares, invokes juror, decision gate
    completeness-reviewer.md     Agent prompt: TBDs, missing criteria, undefined refs
    alignment-reviewer.md        Agent prompt: mockup consistency, standing rules, type contracts
    risk-reviewer.md             Agent prompt: production safety, fail-closed paths, blast radius
    synthesis-agent.md           Opus juror prompt: rules on contested findings only
    project-rules.example.md     Template for project-specific standing rules

README.md
```

## Platform compatibility

The skill content (SKILL.md + companions) is plain markdown describing intent — it works on both Claude Code and Codex. The `.claude-plugin/` and `.codex-plugin/` directories carry platform-specific discovery metadata; the skill logic itself is identical.

## License

MIT
