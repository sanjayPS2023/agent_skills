---
name: implementation-spec
description: Turn the solution-in-memory + repo profile into design sections and a COMPLETE FileMap (create, modify, config, per-env, test and fixture entries) a generator can follow without guessing.
---

# implementation-spec

> Turn the solution-in-memory + repo profile into design sections and a COMPLETE FileMap (create, modify, config, per-env, test and fixture entries) a generator can follow without guessing.

An incomplete FileMap is the most common cause of later "missing artifact" deviations: if a file
is not listed here, generation never writes it and alignment then flags it on every iteration.

### Designers (parallel) -- one file each
- InfrastructureDesignAgent -> `20-spec/Infrastructure-Design.json` (job definition, per-env setup
  files, IAM, retries, timeout, alarms -- `infra` items)
- IntegrationDesignAgent -> `20-spec/Integration-Design.json` (sources and sinks, APIs, S3, config
  files -- `config` items and external contracts)
- BusinessLogicDesignAgent -> `20-spec/BusinessLogic-Design.json` (variant class, rules, validations
  -- `new-code` and `reuse` items)

Each payload carries the section (modules, classes, functions, data structures, in the DETECTED
language's idiom; behaviour only, no code) and a `fileMap` array of entries:
```json
{ "unit": "dcsnet-aggregator", "kind": "source|test|config|properties|setup|test-resource",
  "action": "create | modify", "targetPath": "<ABSOLUTE>",
  "implements": ["BR-...", "EDGE-..."], "reuses": ["<module.function>"],
  "publicApi": ["..."], "dependsOn": ["..."],
  "change": "<for modify: the minimal edit, e.g. 'register DCSNet in the aggregator factory'>",
  "test": { "targetPath": "<ABSOLUTE>" } }
```

### Placing files (read `Repo-Profile.mode`)
- `extend`: new files go INSIDE `Repo-Profile.componentPath`, at the level where the reference keeps that
  kind of file (e.g. `aggregators/<region>/<market>/<variant>/`). Add one `modify` entry for EVERY
  `Repo-Profile.extensionPoints` entry the new variant needs (factory/init module, constants, entrypoint
  routing, enabled-markets in EACH env `.properties`, ...).
- `create`: apply `Repo-Profile.skeleton` with the solution's component name substituted for
  `<COMPONENT>` at every level, rooted at `skeleton.parentPath`; use `fileKindToDir` for the level of
  each kind.
- Path precedence: (1) a path or name stated in `Solution-In-Memory.implementationFiles` / Jira;
  (2) the reference pattern with the solution's names substituted. If they disagree, (1) wins -- record
  the disagreement.
- Per-environment files: one entry per environment in `Repo-Profile.envFiles`, never a subset.
- Never add a prefix the parent lacks, never reuse the sibling's name or domain, never flatten a concern
  the convention nests, never place a file one level up or down from where its kind lives.

### Reuse, not re-implementation
For every Solution-In-Memory item with `realisation: reuse`, list its `reuseTarget` under `reuses` on the
unit that calls it. Do NOT create a new module that duplicates it. New `source` units exist only for
`new-code` items and for the variant's own entry class.

### Tests and fixtures
- Every `source` unit with `action: create` has a `test.targetPath` that follows `Repo-Profile.tests`.
- `config` / `properties` / `setup` entries have `test: null`, but a test that loads them must exist
  when the code reads them.
- Every fixture file a test will open is its own `test-resource` entry under `Repo-Profile.tests.resourcesDir`.

### ASSEMBLY (SpecValidationAgent only -- serial)
Read the three `*-Design.json` + `Solution-In-Memory.json` + `Repo-Profile.json`, then write:
- `FileMap.json`: all entries merged and de-duplicated by `targetPath` (merge their `implements`).
- `Implementation-Spec.json`: the three sections plus a `traceability` table mapping each SIM id ->
  FileMap units, or -> `operational`.

Then FAIL the stage (via `stage-verdict`) when:
- any SIM item with realisation `new-code`/`reuse`/`config`/`infra` maps to no unit;
- an extension point the mode needs has no `modify` entry;
- an environment from `envFiles` is missing;
- a created source unit has no test target;
- a path has an extra prefix segment, a misplaced level or a sibling name.

### Rules
- Design to the profile's conventions and the standards; add no abstraction the requirements do not justify.
- Prefer a small registry/strategy over a long if/elif chain where the solution implies variants.

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
