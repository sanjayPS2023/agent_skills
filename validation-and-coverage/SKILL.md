---
name: validation-and-coverage
description: Triage the Run Validation results (install, static checks, lint, tests, coverage) into environment blockers vs code defects vs coverage gaps, and drive only the repairable ones toward passing tests and the coverage minimum on generated code.
---

# validation-and-coverage

> Triage the Run Validation results (install, static checks, lint, tests, coverage) into environment blockers vs code defects vs coverage gaps, and drive only the repairable ones toward passing tests and the coverage minimum on generated code.

The `Run Validation` script has already run install -> static checks -> lint -> tests + coverage. Its
outputs are in `<RUN ROOT>/50-validation/` (`Validation-Result.json`, `Coverage.json`,
`Static-Checks.json`) and the raw logs are in `<RUN ROOT>/logs/50-validation/` (`install.log`,
`test.log`, `coverage.json`, `lint.log`, `static.log`). Read the RAW LOGS, not only the summary numbers:
the numbers can look normal while the cause is in the log.

### Step 1 -- Is the run even valid?
Check these before reading any coverage number:

| Signal | Meaning |
|---|---|
| `Validation-Result.status = BLOCKED` | Nothing was measured. The `reason` is the finding. |
| `installExit != 0`, or `install.log` contains `ERROR: Failed to build`, `No matching distribution`, `ResolutionImpossible` | Install failed. `pip install -r` is all-or-nothing, so NOTHING from that requirements file was installed. |
| interpreter in `test.log` paths (e.g. `/usr/lib/python3.14/`) differs from `Repo-Profile.pythonVersion` | Wrong runtime. Pinned packages fail to build or import (e.g. `typing.io` gone in 3.13+ breaks pyspark 3.3; pandas 1.5 has no 3.14 wheel). |
| pytest exit `2` with `errors during collection` / `Interrupted` | ZERO tests ran. The coverage number reflects import side-effects only and is meaningless. |
| pytest exit `5` | No tests collected -- wrong directory or naming. |
| pytest exit `4` | Usage error in the test command. |
| pytest exit `1` | Tests ran and some failed -- real results. |

### Step 2 -- Classify every failure
- ENVIRONMENT (not repairable in code): install failures; `ModuleNotFoundError`/`ImportError` for a
  THIRD-PARTY package (pandas, pyspark, boto3, moto, ...) or a stdlib module removed in a newer Python;
  missing java; interpreter/version mismatch. Record each under `environmentBlockers` with the log line
  and the pipeline fix (e.g. "run validation on Python 3.10", "install failed on pandas==1.5.1"). Do
  NOT edit code, tests or dependency pins because of these. When ALL failures are environmental, set
  `status: BLOCKED` in `Fix-Report.json` with `reason: "environment"` so the loop stops instead of
  burning iterations.
- CODE DEFECT (repair): `SyntaxError`; `ImportError`/`NameError`/`AttributeError` on a REPO module
  (a wrong import path, a missing registration, a misspelled name); assertion failures in generated
  tests; static-check errors; lint findings on generated files.
- PRE-EXISTING (do not repair): failures in legacy tests that do not touch generated or modified files.
  Record them under `preExisting` with the test id. Never modify a legacy test.

### Step 3 -- Coverage, scoped correctly
Coverage is judged on the GENERATED and MODIFIED non-test files only (the `Generated-Files.json`
manifest), never on the whole legacy component. From `logs/50-validation/coverage.json` read
`files.<path>.missing_lines` for those files. For each uncovered block, add a test that exercises the
real behaviour behind it (a branch, a boundary, an error path). The minimum is
`Validation-Result.coverageMin` (from `Standards-Checklist.coverageMin`; default 80). If the report
measured the wrong scope, record `coverageScopeWrong: true` -- it is a pipeline defect, not yours to
fix by adding unrelated tests.

### Step 4 -- Re-verify what you changed
Run the affected tests yourself with the same interpreter the validation used
(`Validation-Result.pythonExecutable` when present, else the one matching `Repo-Profile.pythonVersion`).
Record the command, exit code and pass/fail counts in `Fix-Report.json.verification`. If you cannot run
them (environment blocker), say so -- never claim a pass you did not observe.

### Rules
Raise coverage only by exercising real behaviour -- never by deleting assertions, adding `skip`/`xfail`,
`# pragma: no cover`, or excluding files. A failing generated test is a finding to fix in the code
unless the test contradicts Solution-In-Memory, in which case fix the test and cite the SIM id.

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
