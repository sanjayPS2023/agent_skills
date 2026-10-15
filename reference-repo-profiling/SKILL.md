---
name: reference-repo-profiling
description: Locate the reference and target Glue services in the cloned repo, decide create / extend / update mode, and profile them into machine-readable conventions, extension points and the exact toolchain (Glue/Python/Spark versions, test command) every later stage relies on.
---

# reference-repo-profiling

> Locate the reference and target Glue services in the cloned repo, decide create / extend / update mode, and profile them into machine-readable conventions, extension points and the exact toolchain (Glue/Python/Spark versions, test command) every later stage relies on.

The repository is usually a monorepo; services sit several levels down inside sub-packages. Two
services can matter:
- the REFERENCE service -- teaches CONVENTIONS (layout, naming, config shapes, job definition, tests).
  Source: `Run-Inputs.referenceEtlService`, else the component the solution design names
  (`Solution-Model.referenceComponent`).
- the TARGET service (UPDATE only) -- the existing Glue service this run changes.
  Source: `Run-Inputs.targetGlueService`, else the service the solution says to change.
In `update`/`extend` mode the target IS the component being changed; when no separate reference is
given, the target is also the reference.

The reference never overrides the ETL Coding Standards, and never supplies content, names or values for
new work (see the authority order in `workflow-run-contract`).

### Procedure (read the repo on disk with read_file / command_line, absolute paths)
1. LOCATE. For each service name, search the whole repo for a directory whose name is, or ends with, it
   (case-insensitive; also try `-`/`_` swaps). A repo-relative path given in the input is used as is.
   If several match, prefer the one containing a Glue entrypoint (`getResolvedOptions`, `GlueContext`,
   `awsglue`) AND tests. Record `referencePath` and, for update, `targetPath`. Never profile the repo root
   as a fallback: if a service cannot be found, record the searches you ran and set `PARTIAL`.
2. DECIDE THE MODE, constrained by `Run-Inputs.serviceAction`:
   - `CREATE` -> `create`. `UPDATE` -> `extend` or `update`. `AUTO` -> any of the three.
   - `create`: the solution introduces a new, separately deployed Glue job/service.
   - `extend`: the solution adds a new variant (source, market, feed, table, aggregator) to an EXISTING
     job, or says to follow an existing implementation inside the same service.
   - `update`: the solution changes an existing job's behaviour, schema, mapping, config or job
     definition in place.
   Record the sentence of the solution that decided it in `modeEvidence`. If the solution contradicts
   the user's action (e.g. CREATE but the solution only modifies an existing job), keep the user's
   action, set `PARTIAL`, and record the contradiction under `gaps`.
   `componentPath` = the TARGET service for `extend`/`update`, the REFERENCE service for `create`.
3. TOOLCHAIN. Walk UPWARD from `componentPath` to the nearest governing manifest
   (`requirements*.txt`, `pyproject.toml`, `setup.py`, ...). Record:
   - `language`; `governingManifest` (absolute path); `dependencies` (pinned list, verbatim).
   - `glueVersion` from the job's setup/IaC files (`GlueVersion`, `--glue-version`, `glue_version`), as a
     TOP-LEVEL string such as `"4.0"`. The validation script reads this exact key.
   - `pythonVersion` and `sparkVersion` as TOP-LEVEL strings, derived from the Glue version: Glue 5.0 ->
     Python 3.11 / Spark 3.5; Glue 4.0 -> Python 3.10 / Spark 3.3; Glue 3.0 -> Python 3.7 / Spark 3.1.
     For a Python-shell job use the setup file's python version. Generated code and tests must run on
     this version, so it is not optional. If no Glue version is stated anywhere, record `"4.0"`/`"3.10"`
     with `glueVersionSource: "default"` and a gap.
   - `jobType`: `glueetl` (Spark, `awsglue.job.Job`), `pythonshell`, or `gluestreaming`, as the reference
     shows it.
4. TESTS. Record `tests`: `framework`, `testDir` (absolute), `importRoot` (the directory tests are run
   from so the package imports resolve), `testCommand` (pytest arguments run from `importRoot`, e.g.
   `python -m pytest tests -q`), `namingPattern`, `mockingStyle` (unittest.mock, moto, pytest-mock,
   conftest stubs of `awsglue`, local SparkSession fixture), `resourcesDir` (where fixture files live),
   and two representative test files in `referenceFiles`. Use absolute paths.
5. CONVENTIONS (exact shapes, not a summary). For each concern record the representative file AND the
   shape a generator must reproduce:
   - job entrypoint: argument parsing (`getResolvedOptions` keys), GlueContext/Job lifecycle, how config
     is loaded, how the main transform is called, exit/commit handling;
   - transforms: where pure transformation functions live vs I/O (readers/writers), DynamicFrame vs
     DataFrame usage, schema definitions;
   - config: per-variant/per-env config file pattern and its JSON schema (key names, nesting, types);
     `.properties` path, section headers EXACTLY as written (read at runtime -- do not assume a set),
     array-vs-scalar keys, key casing (kebab stays kebab);
   - per-environment files: the COMPLETE list of environments and every file each one has
     (`envFiles`: e.g. `properties/<env>.properties`, `setup/<env>_setup.json`). Later stages must
     produce one per environment, so a partial list causes "missing env config" failures;
   - job definition / IaC: where the Glue job is declared (setup JSON, CloudFormation, Terraform, CDK) and
     its keys (worker type, number of workers, timeout, max retries, `--extra-py-files`,
     `--additional-python-modules`, job bookmarks, connections);
   - logging, exception, constants and registry patterns (each with its file).
6. EXTENSION POINTS (mode `extend` and `update`). For `extend`, pick the existing VARIANT closest to the
   new one (`siblingVariant`) and `grep -rn` its identifier across `componentPath`: every file outside the
   variant's own folder that mentions it is a place a new variant must also be registered (factory/init,
   constants, entrypoint routing, enabled lists in EVERY env `.properties`, job arguments, config). For
   `update`, list every file that implements, configures or tests the behaviour the solution changes
   (grep the field, table, column, rule or job-argument names the solution cites). Record each as
   `{ "file", "line", "what" }` in `extensionPoints`.
7. SKELETON (mode `create`). Record `skeleton` as a PATTERN with a `<COMPONENT>` placeholder:
   `parentPath` (verbatim, no invented prefix), `nesting` (e.g. `<parent>/<COMPONENT>/<COMPONENT>/`),
   `fileKindToDir` (which LEVEL each kind of file lives at: entrypoint, transforms, config, properties,
   setup/job definition, tests, test resources), `subPackageRule` (each concern in its own sub-package --
   the rule, not the reference's domain name).

### Produce
`Reference-Repo-Ref.json` (ingestion -- steps 1-2 and a first look at 3) or `Repo-Profile.json`
(analysis -- every step) payload:
`{ "repo", "branch", "serviceAction", "referenceComponent", "referencePath", "targetComponent",
"targetPath", "componentPath", "mode", "modeEvidence", "siblingVariant", "language", "jobType",
"glueVersion", "glueVersionSource", "pythonVersion", "sparkVersion", "governingManifest", "dependencies",
"layout", "naming", "configResolution", "jobDefinition", "logging", "exceptions", "tests", "envFiles",
"extensionPoints", "skeleton", "conventions", "referenceFiles", "gaps" }`.
Every field cites the file (and line where useful) it was read from. These are patterns to follow, never
names or content to copy into the new work.

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
