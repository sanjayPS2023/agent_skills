---
name: code-generation
description: Write or repair production source, config and registrations at FileMap targets -- reusing existing logic where the solution says so, in the reference's conventions, runnable on the job's Python version, verified before reporting.
---

# code-generation

> Write or repair production source, config and registrations at FileMap targets -- reusing existing logic where the solution says so, in the reference's conventions, runnable on the job's Python version, verified before reporting.

### Inputs
`20-spec/FileMap.json`, `Implementation-Spec.json`, `Solution-In-Memory.json`,
`10-analysis/Repo-Profile.json`, `Standards-Checklist.json`, `00-inputs/Coding-Standards.json`, and
`30-gaps/Gap-Resolutions.json` when it exists.

### Procedure (generation)
1. Take the FileMap entries your prompt assigns you (Scaffold: structure, config, properties, setup,
   job definition, the entrypoint shell and registrations; Core: transforms and business logic;
   Integration: readers, writers, catalog/connection access and clients). Read `glue-job-design` first.
2. `action: create` -> write the file at its ABSOLUTE `targetPath`.
   `action: modify` -> make the MINIMAL in-place edit described in `change`. Preserve every existing
   behaviour, import and test; never reformat or reorder an existing file.
3. `reuses` / SIM `realisation: reuse` -> import and CALL the existing `reuseTarget`. Never copy it,
   re-implement it, or create a parallel module for it.
4. Values: every limit, threshold, path, enum and market id comes from Solution-In-Memory and lives in
   config/constants (standards forbid magic literals). Add constants to the component's existing
   constants module; do not hardcode them in the class.
5. Verify (below), then write your report.

### Follow the reference's CONVENTIONS; take the CONTENT from the solution
Study the files in `Repo-Profile.referenceFiles` and the `siblingVariant` only to LEARN shape: class
pattern (e.g. a `@dataclass` with the same lifecycle methods), config JSON schema, `.properties`
section headers exactly as found, key casing, per-env file set, logging/exception helpers, sub-package
nesting. Do NOT copy the sibling's values, business logic or names. Never write the reference's or
sibling's name or domain into a new file -- not even in a docstring or comment ("mirrors the X
aggregator" is a defect).

### Update mode (`Repo-Profile.mode = update`)
- You are changing a live service. Edit only what a FileMap `modify` entry's `change` names; keep every
  `PRS-*` (preserved) behaviour from Solution-In-Memory intact.
- Keep existing job arguments, config keys and public function signatures backward compatible unless a
  SIM item explicitly changes them; when one does, update every caller and every env file in the same run.
- Never rename, move or reformat existing files; never delete code the SIM does not remove.
- An existing test changes only through a FileMap `modify` entry of kind `test`, and only the assertions
  its cited SIM item changes.

### Runtime correctness
- Target `Repo-Profile.pythonVersion` (e.g. 3.10 for Glue 4.0). Use no syntax or library API newer than that.
- Write every regex as a raw string (`r"\w+"`); `"\w"` is a SyntaxWarning now and an error later.
- Glue: follow `Repo-Profile.jobType` and `glue-job-design`. A `glueetl` entrypoint uses
  `getResolvedOptions`, `job.init(...)` and `job.commit()`. Don't add `Job()` to a component whose
  reference does not use it. Keep transforms as pure functions of DataFrames/records, importable without
  `awsglue`, so tests run without a Glue runtime.
- No credentials, account ids, bucket names or ARNs in source -- use job arguments, config or Secrets
  Manager, as the reference does.
- Never edit a dependency manifest (`requirements*.txt`, `pyproject.toml`) unless the solution
  introduces a new dependency. Changing pins to get an install working is not your call; report it.

### Verify before reporting
- `<python> -m py_compile` on every Python file you wrote or edited, where `<python>` is
  `Validation-Result.pythonExecutable` when it exists, else `<run dir>/.venv-glue/bin/python` when it
  exists (the Baseline Tests script builds it on `Repo-Profile.pythonVersion`), else `python3.10` (or the
  version the profile names). `<run dir>` is the parent of RUN ROOT.
- Run the tests for the units you touched from `Repo-Profile.tests.importRoot`
  (`<python> -m pytest <test files> -q`) when the environment allows it. Record the command and the result.
- Confirm on disk (`test -f`) that every file you claim to have written exists at its absolute path.

### Repair mode (CodeFixAgent)
1. Read `50-validation/Validation-Result.json`, `Static-Checks.json` and `Coverage.json` FIRST and
   triage per `validation-and-coverage` (generated-test failures, regressions against
   `40-codegen/Baseline-Tests.json`, static errors, coverage).
   Environment failures (install failed, wrong interpreter, missing third-party module) are NOT code
   defects: record them and do not change code because of them.
2. Read the previous `Fix-Report.json` (if any) before acting. Do not undo an earlier repair unless the
   current deviation cites a HIGHER authority than the one the earlier repair cited; if two deviations
   conflict, repair neither and record the conflict.
3. Repair in order: missing files / structure -> missing SIM rules (`new-code` first, then `reuse`
   wiring) -> standards -> tests and coverage.
4. A deviation that asks for code for an `operational` or `verifiableIn: none` item is not repairable
   in code: record it as `notRepairable` with the reason.
5. Never weaken, skip or delete a test, and never exclude files from coverage, to get a pass.

### Report
Write the canonical report: ScaffoldAgent -> `Scaffold.json`, CoreImplementationAgent ->
`Core-Implementation.json`, IntegrationImplementationAgent -> `Integration-Implementation.json`,
CodeFixAgent -> `50-validation/Fix-Report.json`. Payload:
`{ "filesWritten": [{ "path": "<repo-relative>", "action": "create|modify", "implements": [ids],
"pathSource": "1|2|3" }], "reuse": [{ "id", "target" }], "verification": [{ "command", "exit",
"summary" }], "deviationsAddressed": [{ "id", "action" }], "notRepairable": [{ "id", "reason" }],
"environmentBlockers": [...], "conflicts": [...] }`.

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
