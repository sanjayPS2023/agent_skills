---
name: implementation-spec
description: Turn the solution-in-memory + repo profile into an implementation spec section and FileMap a generator can follow.
---

# implementation-spec

> Turn the solution-in-memory + repo profile into an implementation spec section and FileMap a generator can follow.

### Produce
Three designers run IN PARALLEL, so each writes its OWN canonical file -- never a shared file (parallel
writes to one file clobber each other). Write your section to EXACTLY:
- InfrastructureDesignAgent -> `Infrastructure-Design.json`
- IntegrationDesignAgent -> `Integration-Design.json`
- BusinessLogicDesignAgent -> `BusinessLogic-Design.json`

Each `*-Design.json` payload carries your section (modules, classes, interfaces, functions, data
structures in the DETECTED language's idiom) AND a `fileMap` array whose entries are
`{ "unit", "targetPath", "publicApi": [...], "dependsOn": [...], "test": { "targetPath" } }`.

`targetPath` and `test.targetPath` are ABSOLUTE and apply `Repo-Profile.skeleton` as a PATTERN with the SOLUTION / run component name substituted for `<COMPONENT>` at every level, rooted at `skeleton.parentPath` -- e.g. `<parentPath>/<NEW>/setup/<env>_setup.json` (OUTER) and `<parentPath>/<NEW>/<NEW>/{main.py, requirements.txt, test/, <subpkg>/...}` (INNER). Use `fileKindToDir` to place each unit at the LEVEL its kind belongs (setup OUTER; entrypoint/manifest/tests/sub-packages INNER). Create a sub-package ONLY for a concern the SOLUTION has, named from the solution (never the sibling's domain name). NEVER add a prefix the parent lacks (no `pace-etl-service`), never the sibling's name, never one level up, nothing flat that the convention nests. The assembler REJECTS (FAIL) any entry with an extra prefix segment, `setup/` not at OUTER, a concern flattened that the convention nests, or a sibling name/domain reused instead of a solution name.

ASSEMBLY (serial, SpecValidationAgent only): read the three `*-Design.json` + `Solution-In-Memory.json`
and WRITE the consolidated `FileMap.json` (all `fileMap` entries merged) and `Implementation-Spec.json`.

### Rules
- Design to the repo profile's conventions; add no abstraction the requirements do not justify.
- Prefer a small strategy/registry over a large if/elif chain where the solution implies interchangeable
  variants. Keep integration/BSP-specific and channel-formatting concerns in separate units.
- Every unit must have a public API and a test target; describe behaviour, do not write code here.

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
