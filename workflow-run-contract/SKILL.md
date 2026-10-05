---
name: workflow-run-contract
description: The run contract every agent obeys -- the on-disk shared-memory layout, the JSON status envelope, and the absolute-path discipline.
---

# workflow-run-contract

> The run contract every agent obeys -- the on-disk shared-memory layout, the JSON status envelope, and the absolute-path discipline.

This is the base skill every agent in the pipeline loads first. It defines HOW agents cooperate; the
task-specific skills define WHAT each one produces.

### Shared memory is the disk
Agents never pass data to each other in memory. The run root you are handed (verbatim -- never search
for it) contains `workflow_output/` with numbered stage folders:

```
workflow_output/
  _run/           run-manifest.json (derived index, never the authority)
  00-inputs/      Solution-Model, Jira-Model, Additional-Instruction, Reference-Repo-Ref, Input-Manifest
  10-analysis/    Business/Functional/NonFunctional/Technical-Requirements, Repo-Profile, Analysis
  20-spec/        Solution-In-Memory, Implementation-Spec, FileMap, Test-Fixtures
  40-codegen/     Scaffold, Core-Implementation, Integration, Tests, Generated-Files
  50-validation/  Toolchain-Profile, Validation-Result, Coverage, Alignment-Result
  90-summary/     Run-Summary
  logs/<stage>/   raw command output
```

Every path already exists (Init Run Workspace created the whole tree). Write ONLY into your own stage
folder; read ONLY files an upstream stage wrote.

### Completeness
A stage is COMPLETE only if its file exists, parses as JSON, and `status` is `OK` or `PARTIAL`.
`BLOCKED`, `SKIPPED`, unparseable, truncated or absent all mean "not done".

### Stage verdict
Every stage ends with a validator writing `<stage-folder>/Stage-Verdict.json`
`{ "stage": "<name>", "verdict": "PASS" | "FAIL", "reasons": [...] }`. That per-stage file is the ONLY
signal Resume reads to decide whether a stage is skipped or re-run. It lives in the stage's own folder.

### Optional inputs
An absent optional input is a recorded fact, never a failure: return `OK`/`PARTIAL` with no fabricated
content.


### Canonical filenames (write and read these BYTE-FOR-BYTE)
Every stage artifact has EXACTLY ONE name. Never a casing, short, or `-report` variant. If a file you
need is absent under its canonical name, it is NOT done -- do not fall back to a differently-named file.

- `00-inputs/` : `Solution-Model.json`, `Jira-Model.json`, `Additional-Instruction.json`,
  `Reference-Repo-Ref.json`, `Solution-Capture-Alignment.json`, `Input-Manifest.json`, `Stage-Verdict.json`
- `10-analysis/` : `Business-Requirements.json`, `Functional-Requirements.json`,
  `NonFunctional-Requirements.json`, `Technical-Requirements.json`, `Repo-Profile.json`, `Analysis.json`,
  `Stage-Verdict.json`
- `20-spec/` : `Solution-In-Memory.json`, `Infrastructure-Design.json`, `Integration-Design.json`,
  `BusinessLogic-Design.json`, `Test-Fixtures.json`, `FileMap.json`, `Implementation-Spec.json`,
  `Stage-Verdict.json`
  (the three `*-Design.json` are the parallel designers' OWN files; `FileMap.json` and
  `Implementation-Spec.json` are assembled by the serial `SpecValidationAgent` from those three.)
- `40-codegen/` : `Scaffold.json`, `Core-Implementation.json`, `Integration-Implementation.json`,
  `Tests.json`, `Generated-Files.json`, `Stage-Verdict.json`
- `50-validation/` : `Toolchain-Profile.json`, `Validation-Result.json`, `Coverage.json`,
  `Alignment-Result.json`, `Fix-Report.json`

Parallel writers never share a file: each parallel agent writes its OWN canonical file; a serial
validator merges them into the consolidated file. This is why casing/short variants are forbidden --
a reader that guesses a name silently loses an upstream stage's work.

## Output envelope (every skill shares this)

Write exactly ONE JSON file to the absolute path your manager/prompt gave you, inside your stage folder. The file MUST have this shape:

```json
{
  "schemaVersion": "1.0",
  "stage": "<00-inputs|10-analysis|20-spec|40-codegen|50-validation>",
  "agent": "<your name>",
  "status": "OK | PARTIAL | BLOCKED | SKIPPED",
  "outputPath": "<the absolute path of this file>",
  "payload": { }
}
```

- `OK` = complete. `PARTIAL` = produced but with recorded gaps. `BLOCKED` = a required skill/tool/input was missing (say which). `SKIPPED` = nothing to do (an absent optional input).
- Put the real CONTENT in `payload`, never a status echo. Push bulky raw output to `logs/<stage>/`.
- Read only files an upstream stage wrote. Never pass data in memory. Never write outside your stage folder.
