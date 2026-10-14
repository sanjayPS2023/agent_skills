---
name: additional-instruction-triage
description: Normalise the free-text run instruction into explicit in-scope items, scope narrowing, source-section focus and constraints.
---

# additional-instruction-triage

> Normalise the free-text run instruction into explicit in-scope items, scope narrowing, source-section focus and constraints.

The additional instruction is authoritative for THIS run's SCOPE. Whatever it names is mandatory; where
it names a subset, everything it does not name is excluded. It can narrow or focus the solution, but it
cannot add behaviour the solution and Jira do not support -- record such a request as a gap.

### Produce
`Additional-Instruction.json` payload:
- `inScope`: explicit items the instruction requires (each with the phrase it came from).
- `narrowsScopeTo`: the named subset (empty if none -> nothing is excluded by it).
- `sourceSections`: document sections the instruction marks as important or exclusive (e.g. "sections
  1 through 11"), as section numbers.
- `constraints`: explicit how-to constraints (e.g. "follow the existing <X> implementation", "no new
  rules"). These feed Solution-In-Memory `decisions`.
- `gaps`: requests not supported by the solution or Jira.
- `raw`: the original text, verbatim.

### Rules
`NA`, empty, null or unresolved `{{template}}` text = ABSENT -> `SKIPPED`, with no invented scope.
Quote; do not paraphrase into broader scope.

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
