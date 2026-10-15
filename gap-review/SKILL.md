---
name: gap-review
description: Collect, de-duplicate and publish the open questions raised by the inputs, analysis and spec stages for a human reviewer, then apply the reviewer's answers to the spec exactly as written before code generation.
---

# gap-review

> Collect, de-duplicate and publish the open questions raised by the inputs, analysis and spec stages for a human reviewer, then apply the reviewer's answers to the spec exactly as written before code generation.

Two agents use this skill: the Gap Presenter (collect) and the Gap Resolution Agent (apply).
Between them, scripts flag HIGH gaps, decide whether a human review is needed, render
`30-gaps/Gaps-For-Review.md` for the reviewer, and stop the run if blocking gaps stay unresolved.

### Part A -- Gap Presenter -> `<RUN ROOT>/30-gaps/Gaps-For-Review.json`
1. Read the `gaps` arrays (and any `PARTIAL` reasons) from EVERY report in `00-inputs/`, `10-analysis/`
   and `20-spec/`, including files whose status is `PARTIAL` -- that is where real gaps live -- and the
   `reasons` of any stage verdict that is `FAIL`.
2. De-duplicate ruthlessly: merge gaps that ask the same question; keep the clearest wording, list every
   raiser in `raisedBy`, keep the highest severity.
3. Drop deployment-time unknowns that are configuration, not questions (account ids, ARNs, bucket names,
   endpoints, secret names) -- record each in `filtered` with the reason; the generated config uses a
   documented placeholder for them.
4. For each remaining gap give a `proposedDefault` the pipeline will use if nobody answers, grounded in
   the solution, the standards or the reference (cite which).
5. Severity: `HIGH` = changes behaviour, data correctness, schema or the job contract; `MED` = changes
   structure or naming; `LOW` = cosmetic. `blocksCodeGeneration: true` only when no defensible default
   exists (e.g. two documented readings with different outputs).
6. ALWAYS write the file, even with zero gaps. Leave every `resolution` empty.

Payload:
```json
{ "summary": { "total": 0, "requiresHumanInput": 0, "blocking": 0 },
  "gaps": [{ "id": "GAP-001", "question": "...", "context": "<file + section / SIM id>",
             "severity": "HIGH|MED|LOW", "raisedBy": ["..."], "proposedDefault": "...",
             "defaultSource": "...", "blocksCodeGeneration": false, "requiresHumanInput": false,
             "resolution": "" }],
  "filtered": [{ "item": "...", "reason": "..." }] }
```
Gap ids are STABLE across re-runs of the same inputs (derive them in source order; on a resumed run keep
the ids of an existing `Gaps-For-Review.json` for the same questions).

### Part B -- Gap Resolution Agent -> `<RUN ROOT>/30-gaps/Gap-Resolutions.json`
Inputs: the published `Gaps-For-Review.json`; the reviewer's uploaded file (a copy of that JSON with
`resolution` filled), and/or the reviewer's typed answers (one per line: `GAP-001: <answer>`; `DEFER`
or blank = deferred); and the `Proceed Without Resolving` flag (`Yes`/`No`). Absent upload and absent
answers are a legitimate state: every gap stays deferred.

1. Apply EXACTLY what the reviewer wrote -- nothing more. When the upload and the typed answers both
   answer the same gap, the typed answer wins (it is the later act); record both.
2. A blank or `DEFER` resolution leaves the gap DEFERRED: its `proposedDefault` applies.
3. An answer that contradicts a document is still the answer: record `override: true` and the document
   location it overrides.
4. `blockingUnresolved` = ids with `blocksCodeGeneration: true` that are still deferred.
   `proceedWithoutResolving` = the flag is `Yes` (case-insensitive; anything else is `No`).
5. Where a resolution changes a spec decision, update the affected `20-spec/` file in place
   (Solution-In-Memory item -- citing the gap id in `source` --, the design section, the FileMap entry,
   Validation-Plan values) so code generation reads the resolved value. List every file you changed.

Payload:
```json
{ "resolved": [{ "id", "resolution", "override": false, "overrides": "<doc location or null>" }],
  "deferred": ["GAP-..."], "proceedWithoutResolving": false, "blockingUnresolved": ["GAP-..."],
  "specFilesUpdated": ["<absolute path>"] }
```

### Rules
Never invent an answer, never resolve a gap the reviewer did not answer, and never delete a gap.

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
