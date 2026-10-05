---
name: additional-instruction-triage
description: Normalise the free-text run instruction into explicit in-scope items and scope narrowing.
---

# additional-instruction-triage

> Normalise the free-text run instruction into explicit in-scope items and scope narrowing.

The additional instruction is authoritative for THIS run. Whatever it names is mandatorily in scope;
where it names a subset, everything it does not name is excluded.

### Produce
`Additional-Instruction.json` payload:
- `inScope`: explicit items the instruction requires.
- `narrowsScopeTo`: the subset named (empty if it names none -> nothing is excluded by it).
- `raw`: the original text.

### Rules
`NA`, empty, null, or unresolved template text = ABSENT -> `SKIPPED`/`OK`, no invented scope.

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
