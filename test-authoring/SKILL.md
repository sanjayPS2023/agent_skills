---
name: test-authoring
description: Author unit tests at their FileMap targets, importing only each unit's public API and asserting behaviour.
---

# test-authoring

> Author unit tests at their FileMap targets, importing only each unit's public API and asserting behaviour.

### Procedure
1. For each unit in `FileMap.json`, write a test at its `test.targetPath`.
2. Import ONLY the unit's `publicApi`. Use `Test-Fixtures.json`.
3. Mirror the reference repo's test framework and layout. Write `Tests.json` listing the tests authored.

### Rules
- Cover the edge cases named in the solution-in-memory (every fallback, limit boundary, and format).
- Where an expected value depends on a reusable pure helper, derive it via that helper rather than a
  magic literal. Tests validate behaviour, not implementation details.
- Include partial-batch / multi-record behaviour where the contract defines it (some succeed, some fail).

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
