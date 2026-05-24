# Multi-Agent Review Skill Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers-extended-cc:subagent-driven-development (recommended) or superpowers-extended-cc:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Ship a global Claude Code skill at `~/.claude/skills/multi-agent-review/` that panels six independent review agents (haiku + sonnet per topic) across three topics, then optionally invokes an opus juror on contested findings, producing a gated Blockers / Warnings / Observations verdict before execution.

**Architecture:** Hub skill (`SKILL.md`) + four companion prompt files. Companion files are read at invocation time and interpolated with artifact content. No code compilation — all files are markdown. Verification is reading the files and doing a live smoke run.

**Tech Stack:** Markdown, Claude Code skill system (`~/.claude/skills/`), Agent tool for parallel dispatch.

---

## File map

| File | Responsibility |
|---|---|
| `~/.claude/skills/multi-agent-review/SKILL.md` | Coordinator: locate artifact, dispatch 6 agents, compare, maybe invoke juror, decision gate |
| `~/.claude/skills/multi-agent-review/completeness-reviewer.md` | Agent prompt: TBDs, missing criteria, undefined references |
| `~/.claude/skills/multi-agent-review/alignment-reviewer.md` | Agent prompt: mockup/codebase/constraint consistency |
| `~/.claude/skills/multi-agent-review/risk-reviewer.md` | Agent prompt: production safety, fail-closed, blast radius |
| `~/.claude/skills/multi-agent-review/synthesis-agent.md` | Opus juror prompt: adjudicate contested findings only |

---

## Task 0: Commit plan + tasks.json

**Goal:** Land the plan and tasks file on `main` so execution can resume across sessions.

**Files:**
- `docs/superpowers/plans/2026-05-24-multi-agent-review-skill.md`
- `docs/superpowers/plans/2026-05-24-multi-agent-review-skill.md.tasks.json`

**Acceptance Criteria:**
- [ ] Both files committed in single commit on `main`
- [ ] Trailer-clean

**Verify:** `git log -1 --format=%s | grep -i 'multi-agent'`

**Steps:**
- [ ] `git add` both files by name
- [ ] Commit. Subject: `docs: multi-agent review skill plan + tasks`
- [ ] Verify trailer-clean: `git log -1 --format='%B' | grep -iE 'co-author|claude' || echo trailer-clean`

```json:metadata
{"files": ["docs/superpowers/plans/2026-05-24-multi-agent-review-skill.md", "docs/superpowers/plans/2026-05-24-multi-agent-review-skill.md.tasks.json"], "verifyCommand": "git log -1 --format=%s | grep -i 'multi-agent'", "acceptanceCriteria": ["Both files committed", "Trailer-clean"]}
```

---

## Task 1: Write `completeness-reviewer.md`

**Goal:** Produce the completeness reviewer companion file with a complete, self-contained agent prompt that any model can follow without additional context.

**Files:**
- Create: `~/.claude/skills/multi-agent-review/completeness-reviewer.md`

**Acceptance Criteria:**
- [ ] File exists at the correct path
- [ ] Contains a fenced prompt block with `[ARTIFACT_CONTENT]` and `[MODE_LABEL]` placeholders that the coordinator interpolates
- [ ] Prompt instructs the model to emit ONLY the structured `FINDINGS:` schema — no prose
- [ ] Covers all five completeness checks from spec §3.2
- [ ] Includes the three severity definitions (BLOCKER / WARNING / OBS)

**Verify:** `cat ~/.claude/skills/multi-agent-review/completeness-reviewer.md | grep -c 'BLOCKER\|WARNING\|OBS'` → ≥ 3

**Steps:**

- [ ] Create the file with this content:

```markdown
# Completeness Reviewer Prompt

Use this file when dispatching the completeness reviewer agent (haiku tier and sonnet tier).
The coordinator reads this file, replaces `[ARTIFACT_CONTENT]` and `[MODE_LABEL]` with
the actual artifact text, then dispatches the agent.

---

## Prompt template

```
You are reviewing a [MODE_LABEL] document for completeness. Your ONLY job is to find
missing, vague, or placeholder content. You are NOT asked to evaluate correctness or style.

## Artifact

[ARTIFACT_CONTENT]

## What to check

1. **Unresolved placeholders** — any literal TBD, TODO, "fill in later", "see below",
   "implementation detail", or clearly incomplete sentence.
2. **Missing acceptance criteria** — any task or requirement that has no concrete,
   independently testable success condition.
3. **Undefined references** — any type, function, endpoint, component, table, or
   CSS class mentioned that is not defined elsewhere in this artifact or clearly
   derivable from an existing file in the codebase.
4. **No verifiable output** — any task that produces no artifact a reviewer could
   inspect (no file path, no test command, no observable behaviour).
5. **What without how** — any requirement that states an outcome but gives no
   implementable direction (e.g. "handle errors appropriately" with no definition
   of what appropriate means).

## Severity definitions

- **BLOCKER** — execution cannot succeed or will produce wrong output without fixing this.
- **WARNING** — likely to cause rework or ambiguity; should fix but not strictly blocking.
- **OBS** — worth noting; low urgency; an attentive implementer would probably catch it.

## Output format — STRICT

Emit ONLY the block below. No introduction, no summary, no prose outside the schema.

FINDINGS:

[BLOCKER|WARNING|OBS] <concise title> | confidence:HIGH|LOW
  detail: <exactly one sentence describing what is missing and where>
  location: <task N / section name / line number if visible>

If you find no issues, emit exactly:
FINDINGS: none
```
```

- [ ] Verify file was created: `ls ~/.claude/skills/multi-agent-review/completeness-reviewer.md`

```json:metadata
{"files": ["~/.claude/skills/multi-agent-review/completeness-reviewer.md"], "verifyCommand": "cat ~/.claude/skills/multi-agent-review/completeness-reviewer.md | grep -c 'BLOCKER'", "acceptanceCriteria": ["File exists", "Contains FINDINGS schema", "Covers all 5 completeness checks", "No prose instructions outside the fenced block"]}
```

---

## Task 2: Write `alignment-reviewer.md`

**Goal:** Produce the alignment reviewer companion file checking consistency against external anchors (mockups, codebase conventions, standing rules, type contracts).

**Files:**
- Create: `~/.claude/skills/multi-agent-review/alignment-reviewer.md`

**Acceptance Criteria:**
- [ ] File exists at the correct path
- [ ] Prompt distinguishes `spec` mode checks from `plan` mode checks
- [ ] Includes standing rules checklist (source-language rule, no inline hex, no Co-Authored-By, canonical constructor)
- [ ] Covers type contract check (readmodel field names vs frontend interface names)
- [ ] Same FINDINGS schema as completeness reviewer

**Verify:** `cat ~/.claude/skills/multi-agent-review/alignment-reviewer.md | grep -c 'spec mode\|plan mode'` → ≥ 2

**Steps:**

- [ ] Create the file with this content:

```markdown
# Alignment Reviewer Prompt

Use this file when dispatching the alignment reviewer agent (haiku tier and sonnet tier).
The coordinator replaces `[ARTIFACT_CONTENT]`, `[MODE_LABEL]`, and `[SUPPLEMENTARY_CONTEXT]`
before dispatching.

`[SUPPLEMENTARY_CONTEXT]` contains:
- spec mode: list of mockup HTML files in docs/design/ and their paths
- plan mode: the linked spec content

---

## Prompt template

```
You are reviewing a [MODE_LABEL] document for alignment with external anchors.
Your ONLY job is to find inconsistencies between this artifact and things that already
exist outside it (mockups, codebase conventions, standing rules, type contracts).

## Artifact

[ARTIFACT_CONTENT]

## Supplementary context

[SUPPLEMENTARY_CONTEXT]

## What to check

### If this is a SPEC:
1. **Mockup alignment** — do all UI element names, CSS class names, and layout
   descriptions in the spec match what the committed mockup HTMLs show?
   Flag any class name in the spec not present in the mockup (or vice versa).
2. **Codebase consistency** — does the spec introduce naming or patterns that
   conflict with how similar things are done in this codebase?

### If this is a PLAN:
1. **Spec coverage** — does every plan task trace back to a stated spec requirement?
   Flag any task with no spec anchor.
2. **Data model consistency** — do field names in readmodel records match the
   corresponding frontend TypeScript interface field names exactly?
3. **Endpoint naming** — do REST paths in the plan match the spec's §11 endpoint list?

### Both modes — standing rules:
4. **Source-language rule** — are there any task titles, step descriptions, or commit
   message suggestions that contain Phase N, T17, M48, "milestone", or "spec §X"?
   These are forbidden in source/tests/commits (plans/specs are exempt).
5. **No inline hex in TSX** — does any plan task instruct adding a hex colour literal
   inside a .tsx or .ts file rather than a CSS class?
6. **No Co-Authored-By** — does any commit message template in the plan include a
   Co-Authored-By trailer?
7. **Record convention** — do any new Java record definitions in the plan omit the
   explicit canonical constructor + `Validators.requireValid(this)` call?

## Severity definitions

- **BLOCKER** — the inconsistency will produce wrong behaviour or silent contract violation.
- **WARNING** — likely rework when the mismatch surfaces; worth fixing now.
- **OBS** — minor drift; an attentive implementer would catch it.

## Output format — STRICT

Emit ONLY the block below. No introduction, no summary, no prose outside the schema.

FINDINGS:

[BLOCKER|WARNING|OBS] <concise title> | confidence:HIGH|LOW
  detail: <exactly one sentence describing the inconsistency and where>
  location: <task N / section name / mockup file>

If you find no issues, emit exactly:
FINDINGS: none
```
```

- [ ] Verify: `ls ~/.claude/skills/multi-agent-review/alignment-reviewer.md`

```json:metadata
{"files": ["~/.claude/skills/multi-agent-review/alignment-reviewer.md"], "verifyCommand": "cat ~/.claude/skills/multi-agent-review/alignment-reviewer.md | grep -c 'spec mode\\|plan mode'", "acceptanceCriteria": ["File exists", "Spec and plan mode checks distinguished", "Standing rules checklist present", "Same FINDINGS schema"]}
```

---

## Task 3: Write `risk-reviewer.md`

**Goal:** Produce the risk reviewer companion file focused on production safety, fail-closed paths, and operational blast radius.

**Files:**
- Create: `~/.claude/skills/multi-agent-review/risk-reviewer.md`

**Acceptance Criteria:**
- [ ] File exists at the correct path
- [ ] Covers all six risk categories from spec §3.4
- [ ] Explicitly calls out trading-path fail-closed as highest-priority
- [ ] Same FINDINGS schema

**Verify:** `cat ~/.claude/skills/multi-agent-review/risk-reviewer.md | grep -c 'fail.closed\|blast radius\|null'` → ≥ 2

**Steps:**

- [ ] Create the file with this content:

```markdown
# Risk Reviewer Prompt

Use this file when dispatching the risk reviewer agent (haiku tier and sonnet tier).
The coordinator replaces `[ARTIFACT_CONTENT]` and `[MODE_LABEL]` before dispatching.

---

## Prompt template

```
You are reviewing a [MODE_LABEL] document for production safety risks and operational
blast radius. Your ONLY job is to find places where failure modes are not handled
fail-closed, or where an error could silently propagate.

This codebase is a live trading system. Trading paths must NEVER fall back to stale
defaults — they must fail loudly. This is the highest-priority concern.

## Artifact

[ARTIFACT_CONTENT]

## What to check

1. **Trading-path fallbacks** — any handler, service, or algorithm that catches an
   exception and returns a default value (e.g. return 1.0 for an FX rate, return
   stale stop price) instead of propagating the error. These are BLOCKER-level.
2. **Missing null / 404 guards** — any endpoint or data fetch that could return null
   for a required field without an explicit guard that either rejects the request or
   returns a structured error response.
3. **Constructor / injection validation** — any new Java class where required
   collaborators are accepted without null-check (Objects.requireNonNull or equivalent).
4. **DB schema changes without migration** — any task that adds/alters a table or
   column without a corresponding Liquibase changeset task.
5. **Nullable → visible text** — any frontend component that could render `undefined`,
   `null`, or `NaN` as literal visible text through implicit string coercion.
6. **Error state omissions** — any user-facing operation in the spec or plan that has
   no stated error state or failure response (what does the UI show if the API call
   fails? what does the backend return if the DB is unavailable?).

## Severity definitions

- **BLOCKER** — could cause silent wrong behaviour in production or data loss.
  Trading-path fallbacks are always BLOCKER.
- **WARNING** — likely to surface as a runtime error or confusing operator experience;
  should be fixed but won't silently corrupt data.
- **OBS** — defensive improvement worth noting.

## Output format — STRICT

Emit ONLY the block below. No introduction, no summary, no prose outside the schema.

FINDINGS:

[BLOCKER|WARNING|OBS] <concise title> | confidence:HIGH|LOW
  detail: <exactly one sentence describing the risk and where>
  location: <task N / section name / component name>

If you find no issues, emit exactly:
FINDINGS: none
```
```

- [ ] Verify: `ls ~/.claude/skills/multi-agent-review/risk-reviewer.md`

```json:metadata
{"files": ["~/.claude/skills/multi-agent-review/risk-reviewer.md"], "verifyCommand": "cat ~/.claude/skills/multi-agent-review/risk-reviewer.md | grep -c 'fail'", "acceptanceCriteria": ["File exists", "Trading-path fallbacks marked BLOCKER", "All 6 risk categories covered", "Same FINDINGS schema"]}
```

---

## Task 4: Write `synthesis-agent.md`

**Goal:** Produce the opus juror companion file. The juror receives all six reports and a pre-compiled contested list, rules only on contested findings, and may synthesise cross-report implications.

**Files:**
- Create: `~/.claude/skills/multi-agent-review/synthesis-agent.md`

**Acceptance Criteria:**
- [ ] File exists
- [ ] Prompt clearly instructs juror NOT to re-review uncontested findings
- [ ] Instructs juror to quote both models verbatim before ruling
- [ ] Allows juror to surface cross-report implications neither model raised alone
- [ ] Same FINDINGS schema for rulings
- [ ] Includes explicit instruction: "You are the final word. Your rulings are binding."

**Verify:** `cat ~/.claude/skills/multi-agent-review/synthesis-agent.md | grep -c 'contested\|uncontested\|binding'` → ≥ 2

**Steps:**

- [ ] Create the file with this content:

```markdown
# Synthesis Agent (Opus Juror) Prompt

Use this file ONLY when the coordinator has detected contested findings.
Replace `[ALL_SIX_REPORTS]` and `[CONTESTED_LIST]` before dispatching.
Model: always opus. This agent is not invoked when all models agree.

---

## Prompt template

```
You are the juror for a multi-model review panel. Six reviewers (haiku and sonnet tiers,
covering completeness, alignment, and risk) have each submitted their findings on the
same artifact. Some findings are contested — the two models in a pair disagreed on
severity, or one model raised a finding the other omitted.

Your job is narrow and specific:
1. Rule on each contested finding.
2. Optionally surface cross-report implications.
3. Do NOT re-review uncontested findings — they are already accepted at stated severity.

You are the final word. Your rulings are binding and override both models' opinions.

## All six reviewer reports

[ALL_SIX_REPORTS]

## Contested findings (your agenda)

[CONTESTED_LIST]

Each entry in the contested list looks like:

  TOPIC: completeness | alignment | risk
  HAIKU said: [BLOCKER|WARNING|OBS|nothing] — <their exact finding text or "not raised">
  SONNET said: [BLOCKER|WARNING|OBS|nothing] — <their exact finding text or "not raised">

## Your task

For each contested finding:

Step 1 — Quote both models' exact language (or note "not raised by <model>").
Step 2 — State your ruling: which model is right, or produce a merged finding.
Step 3 — Emit the ruling in the standard schema.

After ruling on all contested findings, check: do any two UNCONTESTED findings from
different models, read together, imply a third finding that neither model raised?
If so, emit it as an additional finding (mark it SYNTHESISED in the title).

## Output format — STRICT

Emit ONLY the block below.

JUROR RULINGS:

CONTESTED: <original haiku/sonnet titles>
HAIKU: <their exact text or "not raised">
SONNET: <their exact text or "not raised">
RULING: [BLOCKER|WARNING|OBS] <ruling title> | confidence:HIGH
  detail: <one sentence — your ruling with reasoning>
  location: <task N / section>

[repeat for each contested finding]

SYNTHESISED (if any):
[BLOCKER|WARNING|OBS] <title> | confidence:HIGH|LOW
  detail: <one sentence — the cross-report implication>
  location: <source: both reports / topic-A + topic-B>

If no synthesised findings, omit the SYNTHESISED section entirely.
```
```

- [ ] Verify: `ls ~/.claude/skills/multi-agent-review/synthesis-agent.md`

```json:metadata
{"files": ["~/.claude/skills/multi-agent-review/synthesis-agent.md"], "verifyCommand": "cat ~/.claude/skills/multi-agent-review/synthesis-agent.md | grep -c 'contested\\|binding'", "acceptanceCriteria": ["File exists", "Juror told not to re-review uncontested", "Quote-then-rule structure present", "Cross-report synthesis instruction present", "Binding authority stated"]}
```

---

## Task 5: Write `SKILL.md` (coordinator)

**Goal:** Produce the hub skill file with complete coordinator logic, parallel dispatch instructions, comparison algorithm, juror invocation gate, and decision gate. This is the file that gets loaded when `/multi-agent-review` is invoked.

**Files:**
- Create: `~/.claude/skills/multi-agent-review/SKILL.md`

**Acceptance Criteria:**
- [ ] File exists with `name: multi-agent-review` frontmatter
- [ ] Covers both `spec` and `plan` modes
- [ ] Shows exact Agent tool dispatch syntax for all 6 parallel agents
- [ ] Comparison algorithm is explicit: CONTESTED = severity differs OR one model omitted
- [ ] Juror invocation is clearly gated on CONTESTED findings existing
- [ ] Decision gate covers all three outcomes (blockers / warnings / clean)
- [ ] `--fast` flag handling described
- [ ] References all four companion files by name

**Verify:** `cat ~/.claude/skills/multi-agent-review/SKILL.md | grep -c 'completeness-reviewer\|alignment-reviewer\|risk-reviewer\|synthesis-agent'` → 4

**Steps:**

- [ ] Create the file with this content:

````markdown
---
name: multi-agent-review
description: Use before writing-plans (spec mode) or before subagent-driven-development (plan mode). Panels six independent reviewers across three topics, then invokes an opus juror only when models disagree. Produces a Blockers / Warnings / Observations verdict that gates the next workflow step.
---

# Multi-Agent Review

Run a panel of independent reviewers on a spec or plan before execution starts.
Two model tiers (haiku + sonnet) review each of three topics in parallel.
When the tiers disagree, a single opus juror adjudicates.
The verdict gates whether the next workflow step proceeds.

## When to invoke

| Command | Fire after | Proceeds to |
|---|---|---|
| `/multi-agent-review spec` | spec is written and committed | `writing-plans` |
| `/multi-agent-review plan` | plan is written and committed | `subagent-driven-development` |

## Invocation syntax

```
/multi-agent-review [mode] [path?] [--fast]

mode:    spec | plan          (required)
path:    explicit file path   (optional — skip to use most-recent artifact)
--fast:  haiku-only, skip sonnet tier, skip juror (cost-saving for fast iteration)
```

---

## Coordinator steps

### Step 1 — Locate the artifact

**spec mode:**
```bash
ls -t docs/superpowers/specs/*.md | head -1
```
Use the path returned. If `path` arg was provided, use that instead.

Also check for a `docs/design/` subdirectory matching the spec name:
```bash
ls docs/design/ 2>/dev/null
```
If matching design mockups exist, read their filenames — you will pass them to the alignment reviewer as supplementary context.

**plan mode:**
```bash
ls -t docs/superpowers/plans/*.md | grep -v tasks.json | head -1
```
Also find the linked spec: read the plan file, look for `planPath` or a `Spec:` header line, and load that spec as supplementary context for the alignment reviewer.

### Step 2 — Read the artifact

Read the full artifact content into memory. If it is very long (>500 lines), summarise sections that are clearly boilerplate (e.g. repeated header/footer) but keep all task descriptions, data shapes, acceptance criteria, and endpoint definitions verbatim.

### Step 3 — Read companion prompt files

Read these four files from the same directory as this SKILL.md:
- `completeness-reviewer.md`
- `alignment-reviewer.md`
- `risk-reviewer.md`
- `synthesis-agent.md`

Each contains a fenced prompt template. Extract the template (the content inside the outermost ``` fence).

### Step 4 — Build the six agent prompts

For each of the three topic prompts (completeness, alignment, risk):
1. Replace `[ARTIFACT_CONTENT]` with the full artifact text.
2. Replace `[MODE_LABEL]` with `"spec"` or `"plan"`.
3. For alignment reviewer: replace `[SUPPLEMENTARY_CONTEXT]` with either:
   - spec mode: list of mockup HTML filenames + their paths
   - plan mode: the linked spec content

### Step 5 — Dispatch six agents in parallel

Send all six in a single message with parallel Agent tool calls:

```
Agent(completeness-haiku):  model=haiku,  prompt=completeness_prompt
Agent(completeness-sonnet): model=sonnet, prompt=completeness_prompt
Agent(alignment-haiku):     model=haiku,  prompt=alignment_prompt
Agent(alignment-sonnet):    model=sonnet, prompt=alignment_prompt
Agent(risk-haiku):          model=haiku,  prompt=risk_prompt
Agent(risk-sonnet):         model=sonnet, prompt=risk_prompt
```

If `--fast` was passed: dispatch only the three haiku agents and skip to Step 7.

### Step 6 — Compare pairs, identify contested findings

For each topic pair (haiku report vs sonnet report):

A finding is **CONTESTED** if ANY of the following:
- Same finding appears in both reports but at different severity levels
- A finding appears in one report but NOT in the other (by matching title keywords or detail content)
- One model emitted `FINDINGS: none` but the other emitted findings

Build a `CONTESTED_LIST` with the following format for each contested item:
```
TOPIC: completeness | alignment | risk
HAIKU said: [severity] — <exact text> or "not raised"
SONNET said: [severity] — <exact text> or "not raised"
```

### Step 7 — Invoke juror (only if CONTESTED_LIST is non-empty)

If `CONTESTED_LIST` is empty: skip to Step 8.

If `CONTESTED_LIST` has entries:
1. Build the juror prompt from `synthesis-agent.md`:
   - Replace `[ALL_SIX_REPORTS]` with the concatenated raw text of all six reports
   - Replace `[CONTESTED_LIST]` with the contested list built in Step 6
2. Dispatch juror agent: `model=opus`
3. Collect `JUROR RULINGS` response

### Step 8 — Compile verdict

Merge findings from all sources:

```
accepted_findings = []

For each topic pair:
  For each finding where BOTH models agreed (same title + same severity):
    accepted_findings.append(finding at agreed severity)

If juror was invoked:
  For each RULING in JUROR RULINGS:
    accepted_findings.append(ruling)
  For each SYNTHESISED finding (if any):
    accepted_findings.append(synthesised_finding)

BLOCKERS  = [f for f in accepted_findings if f.severity == "BLOCKER"]
WARNINGS  = [f for f in accepted_findings if f.severity == "WARNING"]
OBS       = [f for f in accepted_findings if f.severity == "OBS"]
```

### Step 9 — Decision gate

**If BLOCKERS exist:**

Present blockers clearly:
```
⛔ REVIEW BLOCKED — N blocker(s) must be addressed before proceeding.

BLOCKERS:
1. [title] (topic: X, confidence: Y)
   detail: ...
   location: ...
...
```

Ask the operator:
```
Two options:
  (a) Fix blockers in the artifact now, then I'll re-run the full review.
  (b) Override — acknowledge the blockers and proceed anyway (I'll note the override).
```

**Do NOT auto-proceed.** Wait for operator response.
- If (a): after operator confirms fixes, re-run from Step 1 (full loop, not just juror).
- If (b): log `OVERRIDE: operator acknowledged N blockers` and invoke next skill.

---

**If WARNINGS exist (no blockers):**

Present warnings:
```
⚠ REVIEW PASSED WITH WARNINGS — N warning(s).

WARNINGS:
1. [title] ...
...

Fix warnings before proceeding, or accept and continue?
```

Wait for operator response.
- Fix: re-run from Step 1 after operator confirms.
- Continue: invoke next skill.

---

**If clean (no blockers, no warnings):**

```
✅ Review passed — N findings (0 blockers, 0 warnings, M observations).
```

If OBS > 0, list them below the pass line.

Auto-invoke next skill:
- `spec` mode → invoke `writing-plans`
- `plan` mode → invoke `subagent-driven-development`

---

## Model assignments

| Role | Model | Override |
|---|---|---|
| Fast-tier reviewer | `haiku` | Always used |
| Standard-tier reviewer | `sonnet` | Skipped with `--fast` |
| Juror | `opus` | Not overridable |

## What this skill does NOT do

- Does not repair artifacts — surfaces findings only, operator makes changes
- Does not review code (that is subagent-driven-development's job)
- Does not run on arbitrary files — only specs and plans in `docs/superpowers/`
````

- [ ] Verify companion file references: `cat ~/.claude/skills/multi-agent-review/SKILL.md | grep -c 'completeness-reviewer\|alignment-reviewer\|risk-reviewer\|synthesis-agent'` → 4

```json:metadata
{"files": ["~/.claude/skills/multi-agent-review/SKILL.md"], "verifyCommand": "cat ~/.claude/skills/multi-agent-review/SKILL.md | grep -c 'completeness-reviewer\\|alignment-reviewer\\|risk-reviewer\\|synthesis-agent'", "acceptanceCriteria": ["File exists with correct frontmatter", "Both modes covered", "6 parallel dispatch shown", "Contested comparison algorithm explicit", "Juror gated on CONTESTED_LIST", "Decision gate covers 3 outcomes", "--fast flag described", "All 4 companion files referenced"]}
```

---

## Task 6: Smoke test against a real spec

**Goal:** Invoke `/multi-agent-review spec` against the multi-agent-review spec itself and verify it runs end-to-end: 6 agents fire, comparison runs, decision gate executes.

**Files:** None (read-only smoke run)

**Acceptance Criteria:**
- [ ] All 6 agents return reports (no agent errors)
- [ ] Coordinator correctly identifies any contested findings (or correctly reports none)
- [ ] Decision gate fires and produces a verdict
- [ ] If juror is invoked: it produces JUROR RULINGS in the expected schema
- [ ] Skill auto-proceeds (or stops with blockers/warnings) — does not hang

**Verify:** Run `/multi-agent-review spec docs/superpowers/specs/2026-05-24-multi-agent-review-skill-design.md` in this session.

**Steps:**

- [ ] In the current Claude Code session, run:
  ```
  /multi-agent-review spec docs/superpowers/specs/2026-05-24-multi-agent-review-skill-design.md
  ```
- [ ] Observe: do all 6 agents return without error?
- [ ] Observe: does the coordinator identify contested vs agreed findings?
- [ ] Observe: is the decision gate output readable and correctly formatted?
- [ ] Note any bugs: compare actual output against the coordinator flow in `SKILL.md`
- [ ] If bugs found: fix `SKILL.md` or the companion file and re-run

```json:metadata
{"files": [], "verifyCommand": "/multi-agent-review spec docs/superpowers/specs/2026-05-24-multi-agent-review-skill-design.md", "acceptanceCriteria": ["6 agents return reports", "Contested/agreed comparison runs", "Decision gate produces verdict", "No agent errors"]}
```

---

## Dependencies

```
T0 (commit plan)
T1, T2, T3, T4 (companion files — independent, can run in parallel)
T5 (SKILL.md — requires T1-T4 to exist so it can reference them by name)
T6 (smoke test — requires T5)
```

## Notes

- All skill files go to `~/.claude/skills/multi-agent-review/` — NOT to the repo
- Only the plan `.md` and `.tasks.json` are committed to the repo
- No `Co-Authored-By` trailers
- T1-T4 can be done as a single commit: `git add` is not applicable (these are `~/.claude` files, not in the git repo). No commit needed for skill files themselves.
- If the smoke test (T6) reveals prompt issues, fix the companion file and update the relevant section. No need to re-run T1-T4 if only one companion file needs patching.
