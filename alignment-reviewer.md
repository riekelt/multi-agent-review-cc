# Alignment Reviewer Prompt

Use this file when dispatching the alignment reviewer agent (haiku tier and sonnet tier).
The coordinator replaces `[ARTIFACT_CONTENT]`, `[MODE_LABEL]`, `[SUPPLEMENTARY_CONTEXT]`,
and `[PROJECT_CONTEXT]` before dispatching.

- `[SUPPLEMENTARY_CONTEXT]`: spec mode = mockup file list; plan mode = linked spec content
- `[PROJECT_CONTEXT]`: project-specific standing rules from `project-rules.md` (or empty string if none)

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

## Project-specific standing rules

[PROJECT_CONTEXT]

If [PROJECT_CONTEXT] is empty, skip standing-rules checks (items 4-7 below).

## What to check

### If this is a SPEC:
1. **Mockup alignment** — do all UI element names, CSS class names, and layout
   descriptions in the spec match what the committed mockup files show?
   Flag any class name in the spec not present in the mockup (or vice versa).
2. **Codebase consistency** — does the spec introduce naming or patterns that
   conflict with how similar things are done in this codebase?

### If this is a PLAN:
1. **Spec coverage** — does every plan task trace back to a stated spec requirement?
   Flag any task with no spec anchor.
2. **Data model consistency** — do field names in data models match the
   corresponding interface/type definitions exactly?
3. **Endpoint naming** — do API paths in the plan match the spec's endpoint list?

### Both modes — project standing rules (check only if [PROJECT_CONTEXT] is non-empty):
4-7. Apply each rule listed in [PROJECT_CONTEXT]. For each rule, check whether the
     artifact violates it and flag violations at the severity the rule specifies.

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
