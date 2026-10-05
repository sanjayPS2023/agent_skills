---
name: alignment-verification
description: Diff a produced artifact against its authority and emit an itemised ALIGNED / NOT_ALIGNED verdict.
---

# alignment-verification

> Diff a produced artifact against its authority and emit an itemised ALIGNED / NOT_ALIGNED verdict.

Used in two places: verifying the captured solution against the Confluence page (§6), and verifying the
generated code against the solution-in-memory (§7).

### Procedure
1. Read the AUTHORITY (the source page, or `Solution-In-Memory.json`) and the ARTIFACT (the captured
   model, or the generated code on disk).
2. Walk EVERY item of the authority -- each table/rule/contract/limit/format/decision -- and check the
   artifact honours it.

### Produce
`{ "verdict": "ALIGNED" | "NOT_ALIGNED", "deviations": [{ "item", "expected", "found", "where" }] }`.
`ALIGNED` only when there are zero deviations. Each deviation is concrete enough to fix without re-reading
the authority.

### Rules
Report deviations; do not fix them here (a separate repair/fix agent does). Never mark `ALIGNED` to end a
loop early -- the loop's cap exists precisely so an honest `NOT_ALIGNED` is safe.

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
