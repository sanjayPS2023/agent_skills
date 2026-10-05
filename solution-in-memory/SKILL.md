---
name: solution-in-memory
description: Build the canonical, authoritative solution-in-memory the generated code is checked against.
---

# solution-in-memory

> Build the canonical, authoritative solution-in-memory the generated code is checked against.

This is the single most important design artifact. It is the structured, complete intent that §7's
alignment check diffs the code against.

### Produce
`Solution-In-Memory.json` payload with, derived STRICTLY from `Solution-Model.json` + Jira ACs:
- `businessRules`: every rule, with its exact condition and outcome.
- `contracts`: every input/output contract and field, marked required/optional.
- `limits`: every numeric/format limit, exact.
- `formats`: every supported format/mode and what distinguishes them.
- `decisions`: every explicit "do this, not that" decision, including documented source-document
  disagreements and which reading was chosen and why.
- `edgeCases`: every fallback / graceful-degradation behaviour.

### Rules
Introduce nothing not supported by the solution + ACs. Every item is phrased so code can be checked
against it mechanically. Preserve documented disagreements rather than silently resolving them.

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
