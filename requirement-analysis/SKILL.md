---
name: requirement-analysis
description: Turn captured inputs into one crisp, individually testable slice of the requirements.
---

# requirement-analysis

> Turn captured inputs into one crisp, individually testable slice of the requirements.

Produce ONE requirement slice (business, functional, non-functional, or technical -- your prompt says
which) from the inputs on disk.

### Produce
Write to the EXACT canonical file for your slice -- one of `Business-Requirements.json`, `Functional-Requirements.json`, `NonFunctional-Requirements.json`, `Technical-Requirements.json` (never `Business.json`, `functional-requirements.json`, or any other variant). Write ONLY that one file -- do NOT additionally create a short-named copy. Payload: `requirements: [{ "id", "statement", "source", "acceptance" }]`
where `source` cites the solution/Jira/instruction it came from and `acceptance` is how it will be
checked against generated code.

### Rules
Derive strictly from the sources; add nothing they do not support. Each requirement must be individually
verifiable later by the alignment step. Record unclear points as gaps rather than resolving them by guess.

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
