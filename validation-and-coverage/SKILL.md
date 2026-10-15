---
name: validation-and-coverage
description: Triage the Run Validation results (install, static checks, tests, regressions against the pre-change baseline, coverage) into environment blockers vs code defects vs coverage gaps, and drive only the repairable ones toward passing tests and the coverage minimum on generated code.
---

# validation-and-coverage

> Triage the Run Validation results (install, static checks, tests, regressions against the pre-change baseline, coverage) into environment blockers vs code defects vs coverage gaps, and drive only the repairable ones toward passing tests and the coverage minimum on generated code.

The `Run Validation` script (inside the Code Alignment sub-workflow) has already run, from
`20-spec/Validation-Plan.json`: environment setup (a venv on the job's Python version) -> install ->
static checks -> ONE pytest run over the planned test files plus, in `extend`/`update` mode, the
component's existing suite -> coverage on the generated and modified non-test files. It compares every
failing test with `40-codegen/Baseline-Tests.json` (the same suite run BEFORE any code was generated).

Outputs in `<RUN ROOT>/50-validation/`:
- `Validation-Result.json` payload: `status` reason, `pythonExecutable`, `pythonVersionActual`,
  `interpreterMismatch`, `installExit`, `installFailedPackages`, `testCommand`, `testExit`,
  `tests {collected, passed, failed, errors, skipped}`, `generatedTests {files, failed}`,
  `regressions` (passed at baseline, fail now), `preExistingFailures` (already failing at baseline),
  `unbaselinedFailures` (no baseline available), `coveragePct`, `coverageMin`, `coverageApplicable`,
  `coverageScope`, `staticErrors`, `staticWarnings`.
- `Coverage.json` (per-file percentages and missing lines for the coverage scope) and
  `Static-Checks.json` (`errors` / `warnings`, each `{ file, line, tool, message }`).
Raw logs in `<RUN ROOT>/logs/50-validation/`: `venv.log`, `install.log`, `static.log`, `test.log`,
`junit.xml`, `coverage.json`. Read the RAW LOGS, not only the summary numbers: the numbers can look
normal while the cause is in the log.

The gate PASSES only when: alignment is `ALIGNED`, `status` is not `BLOCKED`, no generated test fails,
`regressions` is empty, `staticErrors` is 0, and coverage meets `coverageMin` (or is not applicable
because no Python source was generated or modified).

### Step 1 -- Is the run even valid?
Check these before reading any coverage number:

| Signal | Meaning |
|---|---|
| `status = BLOCKED` | Nothing trustworthy was measured. The `reason` is the finding. |
| `installExit != 0` and `installFailedPackages` non-empty | Part of the install failed; the script retried package by package. Tests that import those packages cannot run. |
| `toolsAvailable.pytest/coverage/pyflakes = false` | That tool could not be installed in the venv (offline index, blocked package). No pytest = no tests ran; no coverage = `coverageAvailable: false` and the gate reports `coverage-unavailable` -- both ENVIRONMENT, never fixed by editing code. |
| `interpreterMismatch = true` | The venv is not on `Repo-Profile.pythonVersion` (no such interpreter could be provisioned). Version-specific failures are environment, not code. |
| pytest exit `2` with `errors during collection` / `Interrupted` | ZERO tests ran. The coverage number is meaningless. |
| pytest exit `5` | No tests collected -- wrong directory, wrong naming, or no test files written. |
| pytest exit `4` | Usage error in the test command. |
| pytest exit `1` | Tests ran and some failed -- real results; read `generatedTests.failed` and `regressions`. |

### Step 2 -- Classify every failure
- ENVIRONMENT (not repairable in code): install failures; `ModuleNotFoundError`/`ImportError` for a
  THIRD-PARTY package (pyspark, awsglue, pandas, boto3, moto, ...) or a stdlib module removed in a newer
  Python; missing java for a local SparkSession; interpreter/version mismatch. Record each under
  `environmentBlockers` with the log line and the pipeline fix (e.g. "run validation on Python 3.10",
  "install failed on pandas==1.5.1"). Do NOT edit code, tests or dependency pins because of these.
  `awsglue` is never pip-installable: if the generated code imports it at module level in a file the
  tests import, that IS a code defect (transforms must be importable without Glue) unless the reference
  stubs it in `conftest.py` -- then follow the reference. When ALL failures are environmental, set
  `status: BLOCKED` in `Fix-Report.json` with `reason: "environment"` so the loop stops instead of
  burning iterations.
- CODE DEFECT (repair): `SyntaxError`; `ImportError`/`NameError`/`AttributeError` on a REPO module
  (a wrong import path, a missing registration, a misspelled name); failures in generated tests; every
  entry in `regressions` (behaviour that worked before this run and broke); `Static-Checks` errors.
- PRE-EXISTING (do not repair): `preExistingFailures` -- tests that already failed at baseline. Record
  them under `preExisting`. Never modify a legacy test for them.
- `unbaselinedFailures` (no baseline existed): treat like regressions when the failing test imports a
  file this run changed; otherwise record as pre-existing with that reasoning.

### Step 3 -- Coverage, scoped correctly
Coverage is judged on `coverageScope` -- the GENERATED and MODIFIED non-test Python files from
`Generated-Files.json` -- never on the whole legacy component. From `logs/50-validation/coverage.json`
read `files.<path>.missing_lines` for those files. For each uncovered block, add a test that exercises
the real behaviour behind it (a branch, a boundary, an error path). The minimum is `coverageMin`
(from `Standards-Checklist.coverageMin`; default 80). If the scope is wrong (it names files this run did
not touch, or misses ones it did), record `coverageScopeWrong: true` -- a pipeline defect, not yours to
fix by adding unrelated tests.

### Step 4 -- Re-verify what you changed
Run the affected tests yourself with `Validation-Result.pythonExecutable`, from the plan's `importRoot`.
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
