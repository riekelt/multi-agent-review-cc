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
path:    explicit file path   (optional — omit to use most-recent artifact)
--fast:  haiku-only, skip sonnet tier, skip juror (cost-saving for fast iteration)
```

---

## Coordinator steps

### Step 1 — Locate the artifact

**spec mode:**
```bash
ls -t docs/superpowers/specs/*.md | head -1
```
Use the path returned. If a `path` arg was provided, use that instead.

Also check for a matching `docs/design/` subdirectory:
```bash
ls docs/design/ 2>/dev/null
```
If matching design mockups exist, read their filenames — pass them to the alignment reviewer as supplementary context.

**plan mode:**
```bash
ls -t docs/superpowers/plans/*.md | grep -v tasks.json | head -1
```
Also find the linked spec: read the plan file, look for a `Spec:` header line or `planPath`, and load that spec as supplementary context for the alignment reviewer.

### Step 2 — Read the artifact

Read the **full** artifact content verbatim. **Never truncate or summarise any part of the artifact before passing it to agents** — an agent that receives an abbreviated spec will flag missing sections as BLOCKERs, producing false positives that poison the verdict. If the file is large, read it in chunks but assemble the full text before building prompts.

### Step 3 — Resolve model tiers, read companion files, read project rules

**Model tier resolution:**

Read the `"models"` object from the active plugin manifest:
- Claude Code: `.claude-plugin/plugin.json`
- Codex: `.codex-plugin/plugin.json`
- Fallback (no manifest): `{ "fast": "haiku", "standard": "sonnet", "reasoning": "opus" }`

Store three variables — `FAST_TIER`, `STANDARD_TIER`, `REASONING_TIER` — from `models.fast`, `models.standard`, `models.reasoning` respectively. Use these in Step 5 and Step 7 instead of hardcoded model names.

**Read companion prompt files:**

Read these four files from the same directory as this SKILL.md:
- `completeness-reviewer.md`
- `alignment-reviewer.md`
- `risk-reviewer.md`
- `synthesis-agent.md`

Each contains a fenced prompt template. Extract the content inside the outermost ``` fence.

Also check for a `project-rules.md` file in the **project root** (same directory as CLAUDE.md):
```bash
cat project-rules.md 2>/dev/null || echo ""
```
If found, read it into `PROJECT_CONTEXT`. If absent, `PROJECT_CONTEXT` is empty string. This file is where projects configure their standing rules, safety constraints, and codebase-specific conventions.

### Step 4 — Build the six agent prompts

For each of the three topic prompts (completeness, alignment, risk):
1. Replace `[ARTIFACT_CONTENT]` with the full artifact text.
2. Replace `[MODE_LABEL]` with `"spec"` or `"plan"`.
3. Replace `[PROJECT_CONTEXT]` with the content of `project-rules.md` (or empty string).
4. For the alignment reviewer only — replace `[SUPPLEMENTARY_CONTEXT]` with:
   - spec mode: list of mockup HTML filenames + their paths
   - plan mode: the linked spec content

### Step 5 — Dispatch six agents in parallel

Send all six in a **single message** with parallel Agent tool calls:

```
Agent(completeness-fast):     model=FAST_TIER,     prompt=completeness_prompt
Agent(completeness-standard): model=STANDARD_TIER, prompt=completeness_prompt
Agent(alignment-fast):        model=FAST_TIER,     prompt=alignment_prompt
Agent(alignment-standard):    model=STANDARD_TIER, prompt=alignment_prompt
Agent(risk-fast):             model=FAST_TIER,     prompt=risk_prompt
Agent(risk-standard):         model=STANDARD_TIER, prompt=risk_prompt
```

(Substitute `FAST_TIER` / `STANDARD_TIER` with the actual model IDs resolved in Step 3.)

If `--fast` was passed: dispatch only the three `FAST_TIER` agents and skip to Step 7.

⚠ **--fast safety note:** In `--fast` mode, contested findings between haiku reviewers are NEVER adjudicated by the juror. Do NOT use `--fast` on a plan that touches trading paths or production-safety-critical features. Reserve `--fast` for lightweight non-safety specs (tooling, docs, UI copy).

**After dispatching, track which agents returned valid reports.** A valid report contains at least one line starting with `FINDINGS:`. Record which agents errored or timed out — this is used in Step 8's quorum check.

### Step 6 — Compare pairs, identify contested findings

**First — check for errored agents.** If a report is missing or malformed (no `FINDINGS:` line), treat that agent as having returned `FINDINGS: ERROR — agent did not respond`. Proceed to the quorum check in Step 8 before comparing.

For each topic pair (haiku report vs sonnet report for that topic):

A finding is **CONTESTED** if ANY of the following:
- Same finding appears in both reports but at different severity levels
- A finding appears in one report but NOT in the other (match by title keywords or detail content)
- One model emitted `FINDINGS: none` but the other emitted findings

Build a `CONTESTED_LIST` with this format for each contested item:
```
TOPIC: completeness | alignment | risk
HAIKU said: [severity] — <exact text> or "not raised"
SONNET said: [severity] — <exact text> or "not raised"
```

### Step 7 — Invoke juror (ONLY if CONTESTED_LIST is non-empty)

If `CONTESTED_LIST` is empty: skip directly to Step 8.

If `CONTESTED_LIST` has entries:
1. Build the juror prompt from `synthesis-agent.md`:
   - Replace `[ALL_SIX_REPORTS]` with the concatenated raw text of all six reports
   - Replace `[CONTESTED_LIST]` with the contested list built in Step 6
2. Dispatch a single juror agent: `model=REASONING_TIER` (resolved in Step 3)
3. Collect `JUROR RULINGS` response

**Juror failure fallback:** If the juror agent errors, times out, or returns a malformed response (no `JUROR RULINGS:` line), do NOT proceed to Step 8. Instead: promote every contested finding to BLOCKER severity (conservative fallback) and present them to the operator as "JUROR FAILED — treating all contested findings as BLOCKER". This is fail-closed: a dead juror cannot let a contested BLOCKER silently degrade.

### Step 8 — Quorum check + compile final verdict

**Quorum check (do this FIRST):**

Count valid reports (those that returned a `FINDINGS:` line, including `FINDINGS: none`).

```
if valid_reports < 3:
    HALT — present to operator:
    "⛔ REVIEW ABORTED — only N/6 agents returned valid reports.
     Cannot produce a reliable verdict with fewer than 3 reviewers.
     Options: (a) Re-run, (b) Check for API/rate-limit issues, (c) Override and proceed."
    Do NOT invoke the decision gate.

if valid_reports < 6:
    Add a panel-health WARNING to the verdict:
    [WARNING] Partial panel — N/6 agents responded
      detail: <N> of 6 reviewers failed to return a valid report; coverage gaps possible.
      location: Agent dispatch
```

**Then compile accepted findings:**

```
accepted_findings = []

For each topic pair:
  For each finding where BOTH models agreed (same title + same severity):
    add to accepted_findings at agreed severity

If juror was invoked:
  For each RULING in JUROR RULINGS:
    add to accepted_findings
  For each SYNTHESISED finding (if any):
    add to accepted_findings

BLOCKERS  = findings with severity BLOCKER
WARNINGS  = findings with severity WARNING
OBS       = findings with severity OBS
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
```

Ask the operator:
```
Two options:
  (a) Fix blockers in the artifact now, then I'll re-run the full review.
  (b) Override — acknowledge the blockers and proceed anyway (I'll note the override).
```

**Do NOT auto-proceed.** Wait for operator response.
- If (a): after operator confirms fixes, re-run from Step 1 (full loop).
- If (b): write the override record to stdout AND append a `git note` to the artifact's most-recent commit:
  ```bash
  git notes append -m "MULTI-AGENT-REVIEW OVERRIDE $(date -u +%Y-%m-%dT%H:%M:%SZ): operator acknowledged N blockers: [titles]" HEAD
  ```
  Then invoke next skill. The git note is durable and co-located with the artifact's commit history.

---

**If WARNINGS exist (no blockers):**

```
⚠ REVIEW PASSED WITH WARNINGS — N warning(s).

WARNINGS:
1. [title] (topic: X)
   detail: ...
   location: ...

Fix warnings before proceeding, or accept and continue?
```

Wait for operator response.
- Fix → re-run from Step 1 after operator confirms.
- Continue → invoke next skill.

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

## Panel failure semantics

These rules govern what happens when the panel does not complete cleanly:

| Failure | Behaviour |
|---|---|
| < 3 valid reports | **HALT** — abort, present to operator, do not produce verdict |
| 3–5 valid reports | Add panel-health WARNING, continue with partial coverage |
| 6 valid reports | Proceed normally |
| Juror error/timeout | **Conservative fallback** — promote all contested findings to BLOCKER, present as "JUROR FAILED" |
| All agents error | **HALT** — empty accepted_findings must never silently trigger clean verdict |
| `--fast` + contested findings | No juror adjudication — contested findings stay at haiku's severity; operator is warned that adjudication was skipped |

**Never allow an incomplete panel to produce a clean verdict.** If any doubt, halt and present to operator.

---

## Model assignments

Model IDs are resolved at runtime from the active plugin manifest's `"models"` field.
Default Claude Code values shown; Codex and other platforms override via their own plugin.json.

| Tier variable | CC default | Codex default | Role |
|---|---|---|---|
| `FAST_TIER` | `haiku` | `gpt-5.4-mini` | Fast reviewer (always used) |
| `STANDARD_TIER` | `sonnet` | `gpt-5.4` | Standard reviewer (skipped with `--fast`) |
| `REASONING_TIER` | `opus` | `gpt-5.5` | Juror (not overridable — defeats purpose) |

## What this skill does NOT do

- Does not repair artifacts — surfaces findings only; the operator makes changes
- Does not review code — that is `subagent-driven-development`'s job
- Does not run on arbitrary files — only specs and plans in `docs/superpowers/`

## Companion files in this directory

- `completeness-reviewer.md` — checks for placeholders, missing criteria, undefined refs
- `alignment-reviewer.md` — checks against mockups, codebase conventions, standing rules
- `risk-reviewer.md` — checks for production safety, fail-closed paths, blast radius
- `synthesis-agent.md` — opus juror prompt, only dispatched on contested findings
- `design.md` — original design spec for this skill
- `implementation-plan.md` — original implementation plan for this skill
