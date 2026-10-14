---
name: reference-repo-profiling
description: Locate the reference component in the cloned repo, decide create-vs-extend mode, and profile it into machine-readable conventions, extension points and the exact toolchain (Glue/Python/Spark versions, test command) every later stage relies on.
---

# reference-repo-profiling

> Locate the reference component in the cloned repo, decide create-vs-extend mode, and profile it into machine-readable conventions, extension points and the exact toolchain (Glue/Python/Spark versions, test command) every later stage relies on.

The repository is usually a monorepo, and the reference component -- named in the solution design
(`Solution-Model.json`), not by a UI input -- normally sits several levels down inside sub-packages.
Learn its name from the solution, FIND its folder, then profile it.

The reference component shows the CONVENTIONS (layout, naming, shapes) to follow where the ETL Coding
Standards are silent. It does NOT override the standards, and it never supplies content, names or
values for the new work (see the authority order in `workflow-run-contract`).

### Procedure (read the repo on disk with read_file / command_line, absolute paths)
1. LOCATE. Search the whole repo for a directory whose name is, or ends with, the reference component's
   name. If several match, prefer the one that contains source AND tests. Record the absolute path as
   `componentPath`. If the solution names no reference component or none is found, record that and set
   `PARTIAL` -- never profile the repo root as a fallback.
2. DECIDE THE MODE.
   - `extend`: the solution adds a new variant (a new source, market, aggregator, feed, ...) to an
     EXISTING job, or tells you to follow an existing implementation inside the same component. New
     files go INSIDE `componentPath`, and shared modules get minimal in-place edits.
   - `create`: the solution introduces a new, separately deployed component. New files are rooted at
     `skeleton.parentPath`.
   Record the sentence of the solution that decided it in `modeEvidence`.
3. TOOLCHAIN. Walk UPWARD from `componentPath` to the nearest governing manifest
   (`requirements*.txt`, `pyproject.toml`, `setup.py`, `pom.xml`, `package.json`, ...). Record:
   - `language`; `governingManifest` (absolute path); `dependencies` (pinned list, verbatim).
   - `glueVersion` from the job's setup/IaC files (e.g. `GlueVersion`, `--glue-version`), as a TOP-LEVEL
     string such as `"4.0"`. The validation script reads this exact key.
   - `pythonVersion` and `sparkVersion` as TOP-LEVEL strings, derived from the Glue version: Glue 5.0 ->
     Python 3.11 / Spark 3.5; Glue 4.0 -> Python 3.10 / Spark 3.3; Glue 3.0 -> Python 3.7 / Spark 3.1.
     For a Python-shell job use the setup file's python version. Generated code and tests must run on
     this version, so it is not optional.
   - `jobType`: `glueetl` (Spark, uses `awsglue.job.Job`) or `pythonshell`, as the reference shows it.
4. TESTS. Record `tests`: `framework`, `testDir` (absolute), `importRoot` (the directory tests are run
   from so that `import aggregators...` style imports resolve), `testCommand` (e.g.
   `python -m pytest test -q`, run from `importRoot`), `namingPattern`, `mockingStyle` (unittest.mock,
   moto, pytest-mock, conftest stubs of `awsglue`), `resourcesDir` (where fixture files live), and two
   representative test files in `referenceFiles`.
5. CONVENTIONS (exact shapes, not a summary). For each concern record the representative sibling file
   AND the shape a generator must reproduce:
   - config: per-market/variant config file pattern and its JSON schema (key names, nesting, types);
     `.properties` path, section headers EXACTLY as written (read at runtime -- do not assume a set),
     array-vs-scalar keys, key casing (kebab stays kebab);
   - per-environment files: the COMPLETE list of environments and every file each one has
     (`envFiles`: e.g. `properties/<env>.properties`, `setup/<env>_setup.json`). Later stages must
     produce one per environment, so a partial list causes "missing env config" failures;
   - entrypoint, logging, exception, constants and registry patterns (each with its file).
6. EXTENSION POINTS (mode `extend`). Pick an existing sibling VARIANT that is closest to the new one
   (`siblingVariant`), then `grep -rn` its identifier across `componentPath`. Every file outside the
   variant's own folder that mentions it is a place a new variant must also be registered (factory or
   init module, constants, enabled-markets lists in every env `.properties`, routing in the entrypoint,
   config files). Record each as `{ "file", "line", "what" }` in `extensionPoints`.
7. SKELETON (mode `create`). Record `skeleton` as a PATTERN with a `<COMPONENT>` placeholder:
   `parentPath` (verbatim, no invented prefix), `nesting` (e.g. `<parent>/<COMPONENT>/<COMPONENT>/`),
   `fileKindToDir` (which LEVEL each kind of file lives at), `subPackageRule` (each concern in its own
   sub-package -- the rule, not the sibling's domain name).

### Produce
`Reference-Repo-Ref.json` (ingestion) or `Repo-Profile.json` (analysis) payload:
`{ "repo", "branch", "referenceComponent", "componentPath", "mode", "modeEvidence", "siblingVariant",
"language", "jobType", "glueVersion", "pythonVersion", "sparkVersion", "governingManifest",
"dependencies", "layout", "naming", "configResolution", "logging", "exceptions", "tests", "envFiles",
"extensionPoints", "skeleton", "conventions", "referenceFiles" }`.
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
