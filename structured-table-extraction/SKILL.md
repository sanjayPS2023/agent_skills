---
name: structured-table-extraction
description: Turn document tables into faithful row/column JSON so limits, formats and mappings survive intact.
---

# structured-table-extraction

> Turn document tables into faithful row/column JSON so limits, formats and mappings survive intact.

Tables in a solution design usually carry the hard constraints (limits, formats, mappings). Losing a
cell loses a requirement.

### Produce
For each table: `{ "title": "...", "columns": [...], "rows": [[...], ...] }` with cell values copied
EXACTLY (numbers as written, units kept, no rounding, no reordering of rows/columns).

### Rules
- Merge no cells, drop no header, and never "tidy" a value.
- If a table encodes a rule (e.g. a per-format limit), also surface it under the model's `rules` so the
  spec stage cannot miss it. If two tables disagree, record both and their locations.

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
