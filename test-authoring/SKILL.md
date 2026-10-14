---
name: test-authoring
description: Author offline, runnable pytest tests for every generated unit in the reference's style -- asserting each solution rule by behaviour, creating every fixture file they open, and reaching the coverage minimum on the generated code.
---

# test-authoring

> Author offline, runnable pytest tests for every generated unit in the reference's style -- asserting each solution rule by behaviour, creating every fixture file they open, and reaching the coverage minimum on the generated code.

### Inputs
`FileMap.json` (units, `test.targetPath`, `test-resource` entries), `Test-Fixtures.json`,
`Solution-In-Memory.json`, `Repo-Profile.json` (`tests`, `pythonVersion`, `referenceFiles`),
`Standards-Checklist.json` (`coverageMin`, test rules), and the generated source on disk.

### Procedure
1. Read two existing tests from `Repo-Profile.tests.referenceFiles` to learn the import root, mocking
   style (unittest.mock / moto / pytest-mock / conftest stubs), fixture loading and naming. Match them.
2. For each created `source` unit, write its test at `test.targetPath`. Import only the unit's
   `publicApi`, plus the reused modules you need to patch.
3. Create EVERY fixture file a test opens, at its FileMap `test-resource` path, with the content
   `Test-Fixtures.json` defines. When a fixture is derived from a real sample (e.g. an attached schema or
   feed), keep the real structure and field names. Open fixtures relative to the test file
   (`Path(__file__).parent / "resources" / ...`), never via the working directory.
4. Run the tests (below), fix your own test errors, and write `Tests.json`.

### What to assert
- Every `new-code` SIM item: at least one test that asserts its OUTCOME, including each limit boundary
  (just below, at, and just above), each fallback, and partial-batch behaviour (some records pass, some
  are rejected).
- Every `reuse` SIM item: assert the WIRING -- that the new variant calls the `reuseTarget` with the
  variant's own config and passes its result on. Do not re-test the reused function's internals.
- Every `config` item: load the real config file and assert the value and the schema keys the code reads.
- Registration edits: assert the new variant is resolvable through the factory/registry.
- Where the standards require integration-level tests, add a flow test from feed in to sink out, with
  all external boundaries mocked.
- Name each test or its docstring after the SIM ids it covers (e.g. `test_br_price_min_rejects_998`).

### Must run offline on the job's runtime
- No network, no real AWS: patch boto3/S3, HTTP, Elasticsearch, Magento and SNS clients the way the
  reference tests do (moto, `mock.patch`). No sleeps; freeze time where the logic uses it.
- Target `Repo-Profile.pythonVersion`. Write regexes as raw strings.
- Do not start a real Spark session unless the reference tests do; stub `awsglue` in a `conftest.py` only
  if the reference has no such stub and the code imports it.

### Verify
From `Repo-Profile.tests.importRoot`, using the interpreter for `pythonVersion`:
`<python> -m pytest <your test files> -q` and
`<python> -m coverage run --include=<generated non-test files> -m pytest <your test files> -q && <python> -m coverage report`.
- Coverage is measured on the GENERATED and MODIFIED non-test files only; target `coverageMin`
  (default 80).
- If tests cannot be COLLECTED because a third-party package or the interpreter is missing, that is
  an environment blocker: record it and keep the tests as they should be. Do not contort imports to
  dodge it.
- Never mark a test skip/xfail to get a pass.

### Produce
`Tests.json` payload: `{ "tests": [{ "file", "cases": [{ "name", "covers": [ids] }] }],
"fixturesCreated": [...], "uncoveredIds": [...], "run": { "command", "exit", "passed", "failed",
"coveragePct" }, "environmentBlockers": [...] }`.

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
