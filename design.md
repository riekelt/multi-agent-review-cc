# Multi-Agent Review Skill — Design

**Date:** 2026-05-24  
**Status:** Approved for implementation  
**Location:** `~/.claude/skills/multi-agent-review/` (global, applies across all projects)

---

## 1. Purpose

The skill runs a panel of independent review agents on a spec or plan before execution begins. Using multiple model tiers (haiku + sonnet) per topic, it surfaces disagreements that a single reviewer would miss. When models diverge, a single opus juror adjudicates. The output is a unified verdict — Blockers, Warnings, Observations — that gates whether the next workflow step can proceed.

**Integration points:**

| Trigger | Fires after | Auto-proceeds to |
|---|---|---|
| `/multi-agent-review spec` | `brainstorming` spec written | `writing-plans` |
| `/multi-agent-review plan` | `writing-plans` completed | `subagent-driven-development` |

---

## 2. Architecture

### 2.1 File layout

```
~/.claude/skills/multi-agent-review/
  SKILL.md                   — coordinator logic, decision gate, invocation contract
                               (requires YAML frontmatter: name: + description:)
  completeness-reviewer.md   — agent prompt: missing/placeholder/TBD checks
  alignment-reviewer.md      — agent prompt: mockup/codebase/constraint consistency
  risk-reviewer.md           — agent prompt: production safety, blast radius, fail-closed
  synthesis-agent.md         — opus juror prompt: adjudicates contested findings only
```

### 2.2 Agent roster per invocation

Six topic-agents run in **parallel**:

| Agent | Model | Topic |
|---|---|---|
| completeness-haiku | haiku | Completeness |
| completeness-sonnet | sonnet | Completeness |
| alignment-haiku | haiku | Alignment |
| alignment-sonnet | sonnet | Alignment |
| risk-haiku | haiku | Risk |
| risk-sonnet | sonnet | Risk |

The opus juror is **only invoked if contested findings exist** after comparing pairs.

### 2.3 Coordinator flow

```
1. Parse mode arg: "spec" | "plan"
2. Locate artifact:
     spec mode  → most-recently-modified file in docs/superpowers/specs/
     plan mode  → most-recently-modified file in docs/superpowers/plans/
                  + its linked spec (from planPath field or sibling spec)
3. Read artifact content (+ mockup paths if spec mode has a docs/design/ counterpart)
4. Read companion prompt files: completeness, alignment, risk
5. Dispatch 6 agents in parallel, interpolating artifact content into each prompt
6. Collect all 6 reports
7. For each topic pair (haiku vs sonnet):
     - Compare findings: title, severity tag, confidence
     - Mark as CONTESTED if severity differs OR one model omitted the finding
8. If any CONTESTED findings:
     - Read synthesis-agent.md
     - Dispatch opus juror with all 6 reports + contested list
     - Collect juror rulings
9. Compile final verdict:
     - BLOCKERS  = any finding rated BLOCKER (agreed or juror-ruled)
     - WARNINGS  = any finding rated WARNING (agreed or juror-ruled)
     - OBS       = any finding rated OBS (agreed or juror-ruled)
10. Decision gate (§4)
```

---

## 3. Agent prompt specifications

### 3.1 Output schema (all topic agents)

Every reviewer must emit findings in this schema — no prose outside it:

```
FINDINGS:

[BLOCKER|WARNING|OBS] <concise title> | confidence:HIGH|LOW
  detail: <one sentence — what exactly is wrong and where>
  location: <task N / section name / line reference>

[BLOCKER|WARNING|OBS] ...
```

Severity definitions reviewers must apply:

- **BLOCKER** — execution cannot succeed without addressing this. Examples: missing requirement, contradictory spec sections, no acceptance criteria on a production-safety task, undefined type referenced elsewhere.
- **WARNING** — likely to cause rework or runtime issues but could be deferred. Examples: vague criterion, wrong ordering that wastes effort, pattern deviation.
- **OBS** — worth noting, low urgency. Examples: style inconsistency, minor ambiguity with an obvious intent.

### 3.2 Completeness reviewer scope

Check the artifact for:
- Unresolved TBDs, TODOs, "fill in later", placeholder text
- Missing acceptance criteria on any task
- References to types, functions, endpoints, or components not defined anywhere in the artifact or existing codebase
- Tasks that produce no verifiable output (no verify command, no testable criterion)
- Spec: sections that say *what* without saying *how* at a level an implementer can act on

### 3.3 Alignment reviewer scope

Check the artifact against external anchors:
- **Spec mode**: does the spec align with committed mockup HTMLs in `docs/design/`? Are all mockup elements named and accounted for? Are CSS class names consistent with what `strategy-insight.css` defines?
- **Plan mode**: does every plan task trace back to a spec requirement? Are there tasks with no spec backing? Does the plan's data model match the spec's data model?
- Standing rules compliance: source-language rule (no `Phase N`, `T17` etc.), no inline hex in TSX, no `Co-Authored-By` trailer mentioned anywhere, records use canonical constructor + `Validators.requireValid`.
- Backend/frontend type contracts: readmodel fields match frontend interface field names.

### 3.4 Risk reviewer scope

Check for production safety and operational blast radius:
- Trading paths that fall back to stale/default values instead of failing closed
- Missing 404/null-guard on endpoints that serve live data
- Handler constructors that accept null for required collaborators without validation
- Tasks touching DB schema without a Liquibase changeset
- Frontend components that could render `undefined` as visible text (e.g. raw `.toString()` on nullable)
- Spec: no explicit mention of error state for any user-facing operation that can fail

### 3.5 Synthesis (juror) scope

The juror reads all 6 reports and the contested list. For each contested finding:

1. Quote both models' exact finding text
2. State which model is correct and why, or declare both partially right with a merged ruling
3. Emit the ruling in the standard schema (BLOCKER/WARNING/OBS + detail + location)

The juror does not re-review uncontested findings — it rules only on disagreements. It may also flag if two models' *separate* uncontested findings together imply a third finding neither raised.

---

## 4. Decision gate

```
if BLOCKERS:
    Present blockers to operator.
    Offer two paths:
      (a) Fix blockers now — operator updates artifact, re-run full /multi-agent-review loop
      (b) Override with note — operator acknowledges and proceeds anyway (logged)
    Skill does NOT auto-proceed past blockers.

elif WARNINGS:
    Present warnings.
    Ask: "Fix before proceeding, or accept and continue?"
    If fix: operator updates, re-run full loop
    If continue: invoke next skill (writing-plans or subagent-driven-development)

else:
    Print: "Review passed — N findings (0 blockers, 0 warnings, M observations)."
    If any OBS: list them below the pass line.
    Auto-invoke next skill.
```

Re-run on blocker fix is a **full loop** (all 6 agents), not just the juror, because fixing a blocker may introduce new issues.

**Override log:** BLOCKER overrides are written to stdout AND appended as a `git notes` entry on the artifact's most-recent commit for durable, co-located audit trail.

---

## 5. Invocation contract

```
/multi-agent-review [mode] [path?] [--fast]

mode:    spec | plan          (required)
path:    explicit file path   (optional — defaults to most-recently-modified artifact)
--fast:  haiku-only, skip sonnet tier, skip juror (cost-saving; NOT safe for trading-path plans)
```

Examples:
```
/multi-agent-review spec
/multi-agent-review plan
/multi-agent-review spec docs/superpowers/specs/2026-05-24-my-feature-design.md
```

Model overrides via args (for cost control in CI or fast iterations):
```
/multi-agent-review plan --fast      # haiku-only, skip sonnet, skip juror
/multi-agent-review plan --thorough  # add a third sonnet pass per topic
```

---

## 6. Model assignments (defaults)

| Role | Default | Override flag |
|---|---|---|
| Topic agents (fast tier) | `haiku` | `--fast` skips sonnet |
| Topic agents (standard tier) | `sonnet` | `--fast` uses haiku only |
| Juror | `opus` | Not overridable (defeats purpose) |

---

## 7. Panel failure semantics

| Failure | Behaviour |
|---|---|
| < 3 valid reports | **HALT** — abort, present to operator, no verdict |
| 3–5 valid reports | Add panel-health WARNING, continue with partial coverage |
| 6 valid reports | Proceed normally |
| Juror error/timeout | **Conservative fallback** — all contested findings → BLOCKER; present as "JUROR FAILED" |
| All agents error | **HALT** — empty panel must never silently trigger clean verdict |
| `--fast` + contested | No juror; operator warned adjudication was skipped |

Never allow incomplete panel to produce a clean verdict.

---

## 8. Scope boundaries (was §7)

**In scope:**
- Spec and plan review as defined above
- Global skill usable in any project
- Decision gate with blocker/warning/observation taxonomy
- Juror invoked only on disagreements

**Out of scope (deliberate exclusions):**
- Post-execution code review (that is `subagent-driven-development`'s spec-reviewer + code-quality-reviewer)
- Running against arbitrary files outside the spec/plan workflow
- Automatic artifact repair (skill surfaces findings; it does not rewrite the spec/plan itself)
- CI/CD integration — operator-invoked only

---

## 9. Self-review notes (was §8)

- No TBDs in any section
- Decision gate covers all three outcome states
- Juror-only-on-disagreement is explicit in §2.3 step 8
- `--fast` override prevents cost blowup during fast iteration
- Out-of-scope boundary prevents scope creep into code review territory
