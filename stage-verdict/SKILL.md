---
name: stage-verdict
description: Write a stage's Stage-Verdict.json -- the single PASS/FAIL signal Resume reads -- using the stage-specific completeness checks.
---

# stage-verdict

> Write a stage's Stage-Verdict.json -- the single PASS/FAIL signal Resume reads -- using the stage-specific completeness checks.

You run last in your stage. Judge only what is on disk.

### Procedure
1. Read every expected file of the stage under its canonical name (see `workflow-run-contract`). A file
   that is missing, unparseable, or `BLOCKED` (or `SKIPPED` when it was required) means FAIL.
2. Apply the stage's checks below.
3. Write `<RUN ROOT>/<stage>/Stage-Verdict.json`.

### Stage checks
- 00-inputs (InputValidationAgent, which runs AFTER the solution-capture loop):
  `Run-Inputs.json` exists; Solution-Model captured with its tables and rules (or `SKIPPED` because no
  page was given -- then FAIL only if `serviceAction` and the Jira ACs give nothing to build);
  Jira-Model has every requested key or a recorded fetch error; Coding-Standards transcribed when
  `codingStandardsFile` is set (a supplied but unreadable file is FAIL); Reference-Repo-Ref has a real
  `componentPath` that exists on disk (`test -d`); Solution-Capture-Alignment is `ALIGNED`, or every
  remaining deviation is recorded as a gap in Solution-Model.
  A fetch failure for a PRESENT required input is FAIL; an ABSENT optional input is never FAIL.
- 10-analysis: every requirement file present with ids and a `source` for each; Standards-Checklist has
  severities and a numeric-or-null `coverageMin`; Repo-Profile has `mode` (consistent with
  `Run-Inputs.serviceAction`), `componentPath` (exists), `glueVersion`, `pythonVersion`,
  `tests.importRoot`, `tests.testCommand`, a non-empty `envFiles`, `extensionPoints` when `mode` is
  `extend` or `update`, and `skeleton` when `mode` is `create`.
- 20-spec (SpecValidationAgent, after the assembly described in `implementation-spec`): every SIM item
  has a `realisation`; every non-operational item traces to at least one FileMap unit; every environment
  and every needed extension point has an entry; every created source unit has a test target; no
  path has an extra prefix, a wrong level or a reference/sibling name; `Validation-Plan.json` exists with
  absolute `componentDir` and `importRoot` and a `pythonVersion`.
- 40-codegen: written by the gate SCRIPT, never by an agent.

### Produce
`Stage-Verdict.json` = `{ "stage": "<folder name>", "verdict": "PASS" | "FAIL", "reasons": [...] }` --
flat, WITHOUT the status envelope (the gate scripts read `verdict` at the top level). On `FAIL`, list
itemised, actionable reasons (file + what is missing) so the next iteration knows what to fix.

### Rules
Never fabricate a `PASS`: this file drives Resume, and a wrong `PASS` silently skips real work. Never
`FAIL` for an absent OPTIONAL input.

## Output envelope (every skill shares this)

Write exactly ONE report JSON at its canonical name inside your stage folder, using an ABSOLUTE path
(`<RUN ROOT>/<stage>/<CanonicalName>.json`). The file MUST have this shape:

```json
{
  "schemaVersion": "1.0",
  "stage": "<00-inputs|10-analysis|20-spec|30-gaps|40-codegen|50-validation|90-summary>",
  "agent": "<your name>",
  "status": "OK | PARTIAL | BLOCKED | SKIPPED",
  "outputPath": "<the absolute path of this file>",
  "payload": { }
}
```

- `OK` = complete. `PARTIAL` = produced but with recorded gaps. `BLOCKED` = a required skill/tool/input
  was missing (say exactly which). `SKIPPED` = nothing to do (an absent optional input).
- Put the real CONTENT in `payload`, never a status echo. Push bulky raw output to `<RUN ROOT>/logs/<stage>/`.
- Read only files an upstream stage wrote. Never pass data in memory.
- Write boundaries: your report goes ONLY in your own stage folder. Agents that generate code
  (Scaffold, CoreImplementation, IntegrationImplementation, TestAuthoring, CodeFix) ALSO write source,
  config and test files -- but only inside the repo, under `Repo-Profile.componentPath` or at a path the
  FileMap names, never under RUN ROOT. Exception: `Stage-Verdict.json` uses its own flat shape (see
  `stage-verdict`).
