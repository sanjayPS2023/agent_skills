---
name: workflow-run-contract
description: The run contract every agent obeys -- run-root resolution, the on-disk stage layout and canonical filenames, the authority order, write boundaries, and the JSON status envelope.
---

# workflow-run-contract

> The run contract every agent obeys -- run-root resolution, the on-disk stage layout and canonical filenames, the authority order, write boundaries, and the JSON status envelope.

This is the base skill every agent loads first. It defines HOW agents cooperate; task skills define WHAT
each one produces.

### Skills are on disk
The skill blocks in your prompt are names and one-line summaries only. Read
`<Skills Directory>/<skill-name>/SKILL.md` for every skill you rely on BEFORE acting. If a required
SKILL.md cannot be read, return `BLOCKED` naming that skill -- never improvise its procedure.

### RUN ROOT
RUN ROOT is the `workflow_output` directory itself: the folder that DIRECTLY contains the numbered
stage folders. Every `<RUN ROOT>/<stage>/<file>` means exactly that path -- never
`<repo>/<stage>/...` and never `<RUN ROOT>/workflow_output/<stage>/...`.
- Use the RUN ROOT your prompt gives you verbatim. Never search for it or guess it.
- If the prompt still shows an unresolved template (`{{...}}`), RUN ROOT is the `workflow_output` folder
  that sits next to the `.agents` folder named in `Skills Directory:`.
- A "file not found" on a path you built any other way is NOT evidence that a stage has not run. Re-check
  from RUN ROOT before concluding anything.

The repository being changed is the git working tree next to RUN ROOT (normally `<run dir>/src`).
Repo-relative paths in reports are relative to that repo root. Always READ and WRITE with ABSOLUTE paths:
your shell's working directory is NOT the repo, so a relative write lands outside it and the file is
silently lost from the manifest, the tests and the commit.

### Stage layout
```
<RUN ROOT>/
  _run/           run-manifest.json, resume-plan.json (derived indexes, never the authority)
  00-inputs/      captured inputs + the staged coding standards (source/)
  10-analysis/    requirement slices, standards checklist, repo profile
  20-spec/        solution-in-memory, designs, FileMap, implementation spec, test fixtures
  30-gaps/        gaps for human review and the reviewer's resolutions
  40-codegen/     generation reports, generated-files manifest, stage verdict
  50-validation/  validation/coverage/static results (scripts), alignment result, fix report
  90-summary/     run summary
  logs/<stage>/   raw command output (install.log, test.log, coverage.json, ...)
```
Init Run Workspace creates every folder before any agent runs.

### Canonical filenames (write and read these BYTE-FOR-BYTE)
Each artifact has exactly ONE name. Never a casing, short or `-report` variant. If a file you need is
absent under its canonical name, it does not exist -- do not fall back to a similarly-named file.

| Folder | File | Written by |
|---|---|---|
| 00-inputs | `Solution-Model.json` | SolutionModelAgent |
| 00-inputs | `Jira-Model.json` | JiraModelAgent |
| 00-inputs | `Additional-Instruction.json` | AdditionalInstructionAgent |
| 00-inputs | `Coding-Standards.json` | CodingStandardsAgent |
| 00-inputs | `Reference-Repo-Ref.json` | ReferenceRepoAgent |
| 00-inputs | `Solution-Capture-Alignment.json` | SolutionCaptureVerifyAgent |
| 00-inputs | `Input-Manifest.json` | Input Ingestion Manager |
| 10-analysis | `Business-Requirements.json`, `Functional-Requirements.json`, `NonFunctional-Requirements.json`, `Technical-Requirements.json`, `Data-Mapping-Requirements.json` | one requirement agent each |
| 10-analysis | `Standards-Checklist.json` | StandardsChecklistAgent |
| 10-analysis | `Repo-Profile.json` | RepositoryDiscoveryAgent |
| 10-analysis | `Analysis.json` | Requirement Analysis Orchestrator |
| 20-spec | `Solution-In-Memory.json` | SolutionInMemoryAgent |
| 20-spec | `Infrastructure-Design.json`, `Integration-Design.json`, `BusinessLogic-Design.json` | one designer each (parallel) |
| 20-spec | `Test-Fixtures.json` | TestFixtureAgent |
| 20-spec | `FileMap.json`, `Implementation-Spec.json` | SpecValidationAgent (serial assembler) |
| 30-gaps | `Gaps-For-Review.json` | Gap Presenter |
| 30-gaps | `Gap-Resolutions.json` | Gap Resolution Agent |
| 40-codegen | `Scaffold.json`, `Core-Implementation.json`, `Integration-Implementation.json`, `Tests.json` | one generator each |
| 40-codegen | `Generated-Files.json` | manifest SCRIPT (derived from git; never hand-written) |
| 50-validation | `Validation-Result.json`, `Coverage.json`, `Static-Checks.json` | Run Validation SCRIPT |
| 50-validation | `Alignment-Result.json` | CodeSolutionAlignmentAgent |
| 50-validation | `Fix-Report.json` | CodeFixAgent |
| 90-summary | `Run-Summary.json` | Run Summary script |
| every stage | `Stage-Verdict.json` | that stage's validator (for 40-codegen: the gate SCRIPT) |

Parallel writers never share a file; a serial assembler merges their files. Never write a file a
script owns (`Generated-Files.json`, `Validation-Result.json`, `Coverage.json`, `Static-Checks.json`,
`40-codegen/Stage-Verdict.json`).

### Authority order (what wins when sources disagree)
WHAT to build:
1. Solution design (`Solution-Model.json`) + Jira acceptance criteria (`Jira-Model.json`), as AMENDED by
   reviewer decisions in `30-gaps/Gap-Resolutions.json` (a resolution overrides the default it resolves)
   and NARROWED by `Additional-Instruction.json` (it can restrict scope, never add unsupported scope).

HOW to build it:
2. ETL Coding Standards (`Coding-Standards.json` / `Standards-Checklist.json`) -- MANDATORY rules are
   binding; RECOMMENDED rules are warnings.
3. Reference component conventions (`Repo-Profile.json`) -- layout, naming and shapes where the
   standards are silent.
4. Skill defaults.

When two sources at the same level disagree, record both readings and the location; do not silently pick.

### Completeness
A stage is COMPLETE only if its file exists, parses as JSON, and `status` is `OK` or `PARTIAL`.
`BLOCKED`, `SKIPPED` (when the file was required), unparseable, truncated or absent all mean "not done".

### Stage verdict
Each stage ends with `<stage-folder>/Stage-Verdict.json` = `{ "stage", "verdict": "PASS"|"FAIL",
"reasons": [...] }`. That per-stage file is the ONLY signal Resume reads. It has NO envelope.

### Optional inputs
An absent optional input is a recorded fact, never a failure: return `OK`/`SKIPPED` with no fabricated
content. `NA`, empty, null and unresolved `{{template}}` text all mean ABSENT.

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
