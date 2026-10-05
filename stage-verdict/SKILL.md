---
name: stage-verdict
description: Write a stage's Stage-Verdict.json -- the single PASS/FAIL signal Resume reads.
---

# stage-verdict

> Write a stage's Stage-Verdict.json -- the single PASS/FAIL signal Resume reads.

You run last in your stage. Decide whether the stage's output is complete and correct.

### Procedure
1. Read every expected file in the stage folder. A file that is missing, unparseable, or whose `status`
   is `BLOCKED`/`SKIPPED` (when it was required) means the stage is not done.
2. Check the stage's specific completeness rules (e.g. every required input captured; every unit in the
   FileMap has a test target).

### Produce
`<stage-folder>/Stage-Verdict.json` = `{ "stage": "<name>", "verdict": "PASS" | "FAIL",
"reasons": [...] }`. `PASS` only when the stage's output is complete AND correct. On `FAIL`, list
itemised reasons so the next iteration knows what to fix.

### Rules
Judge only what is on disk. Never fabricate a `PASS`. This file drives Resume: a wrong `PASS` silently
skips real work.

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
