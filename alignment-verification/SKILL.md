---
name: alignment-verification
description: Diff a produced artifact against its authority and emit an itemised, evidence-backed, iteration-stable ALIGNED / NOT_ALIGNED verdict -- reuse-aware, scoped to what this run changed, with no contradictory or unverified deviations.
---

# alignment-verification

> Diff a produced artifact against its authority and emit an itemised, evidence-backed, iteration-stable ALIGNED / NOT_ALIGNED verdict -- reuse-aware, scoped to what this run changed, with no contradictory or unverified deviations.

Used in two places:
- SOLUTION CAPTURE (SolutionCaptureVerifyAgent): the captured `Solution-Model.json` against the source
  page. Output: `00-inputs/Solution-Capture-Alignment.json`.
- CODE (CodeSolutionAlignmentAgent): the generated code on disk against `Solution-In-Memory.json`, the
  ETL Coding Standards and the reference layout. Output: `50-validation/Alignment-Result.json`.

Your verdict drives a repair loop. A deviation that is unverified, out of scope, contradictory or
re-worded between iterations makes the loop oscillate instead of converge. Precision beats volume.

### Procedure (code alignment)
1. Read the authorities: `Solution-In-Memory.json`, `Jira-Model.json` ACs, `Gap-Resolutions.json` (if
   present), `Additional-Instruction.json`, `Standards-Checklist.json`, `Coding-Standards.json`,
   `Repo-Profile.json` (note its `mode`), `FileMap.json`, and the `glue-job-design` rules.
   Also read `50-validation/Validation-Result.json` for `regressions` and `generatedTests.failed`.
2. Read what this run changed: `40-codegen/Generated-Files.json`, confirmed against the real tree with
   `git -C <repo> status --porcelain` / `ls`.
3. Read the PREVIOUS `Alignment-Result.json` and `Fix-Report.json` if they exist. For each previous
   deviation, record `resolved` or `open` with fresh evidence. Re-use the same id for the same issue.
4. Walk every SIM item, standards check and structural rule, and judge it as below.
5. Write the result.

### How to judge each item
- `realisation: new-code` -> find the implementing code. Cite `file:line` when present; deviation when absent.
- `realisation: reuse` -> SATISFIED when the new code calls `reuseTarget`, directly or through a call
  path you can cite hop by hop as `file:line`. Never demand a re-implementation of reused logic.
- `realisation: config` -> check the value in the config/properties file(s), in every environment where
  it applies.
- `realisation: infra` -> check the setup/job-definition files (every environment).
- `changeType: change|remove` (update mode) -> the code at `currentBehaviour` no longer does the old
  thing and now does the new one; cite both. An untouched old implementation is `missing-implementation`.
- `preserved` (`PRS-*`) -> still behaves as before. Each entry in `Validation-Result.regressions` is a
  `blocking` deviation of kind `regression` naming the failing test and the PRS/SIM id it protects.
- Glue rules (`glue-job-design` checklist) -> evaluate on the files this run created or modified.
- `realisation: operational` / `verifiableIn: none` -> never a deviation. List it under `notVerifiable`.
- Standards -> evaluate each applicable `Standards-Checklist` item with its `verification` method, ONLY on
  files this run created or modified (unless the item is `appliesTo: component`). Legacy code is out of
  scope. Failed MANDATORY = `blocking`; failed RECOMMENDED = `warning`.
- Structure -> every `envFiles` environment has its file; every `extensionPoints` registration needed by
  the mode is present; paths sit at the reference's level with no extra prefix; the reference's or
  sibling's name/domain does not appear in any new file (code, docstring or comment). In `update` mode
  the target service's own name is of course allowed; only a SEPARATE reference service's name leaks.

### Evidence rules (before writing a deviation)
- "Missing file": prove it with `test -f <absolute path>` (exit 1). Never report a file missing that
  `Generated-Files.json` lists or the tree contains.
- "Missing implementation": state where you looked (files and grep pattern).
- One deviation per (rule, file). Never emit two deviations that demand opposite things (e.g. "use
  `@dataclass`" and "do not use `@dataclass()`") -- resolve against the reference file and emit at most one.
- Resolve each missing artifact's expected path ONCE, with precedence: (1) a path or name stated by SIM
  or Jira; (2) a FileMap entry; (3) the reference pattern with the solution's names. Never give two
  names for the same artifact, and never `N/A` as a file.

### Produce
```json
{ "verdict": "ALIGNED | NOT_ALIGNED",
  "deviations": [{ "id": "DEV-<ruleId>-<file-stem>", "kind": "missing-file|missing-implementation|missing-registration|structural-parity|standards|reference-name-leak|config-value|regression|glue-runtime",
                   "severity": "blocking|warning", "file": "<repo-relative>", "line": 0,
                   "ruleId": "<SIM or standards id>", "detail": "<one-line summary>", "expected": "...", "found": "...", "evidence": "<command or file:line>",
                   "previously": "new|open" }],
  "resolvedSincePrevious": ["DEV-..."],
  "notVerifiable": ["EDGE-016", "..."],
  "standardsCompliance": { "checked": 0, "passed": [], "failed": [], "notApplicable": [], "warnings": [] } }
```
`verdict` is `NOT_ALIGNED` if and only if at least one `blocking` deviation exists. Warnings never block.
Each deviation must be fixable without re-reading the authority.

### Solution-capture alignment
Same shape. The authority is the source page; the artifact is `Solution-Model.json`. Each deviation
names the page section, the exact expected value (table cell, limit, rule text) and what was captured.

### Rules
Report; never fix code here. Never mark `ALIGNED` to end a loop early, and never invent a deviation to
keep it going. The loop cap makes an honest `NOT_ALIGNED` safe.

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
