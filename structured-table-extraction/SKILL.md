---
name: structured-table-extraction
description: Turn document tables into faithful row/column JSON so limits, formats, mappings and status markers survive intact.
---

# structured-table-extraction

> Turn document tables into faithful row/column JSON so limits, formats, mappings and status markers survive intact.

Tables in a solution design or standards document usually carry the hard constraints (limits, formats,
field mappings, "already implemented" status columns). Losing a cell loses a requirement.

### Produce
For each table: `{ "title", "section", "columns": [...], "rows": [[...], ...] }`, with cell values
copied EXACTLY: numbers as written, units kept, no rounding, no reordering of rows or columns, and empty
cells as `""` (never dropped, which would shift columns).

### Rules
- Merge no cells, drop no header, never "tidy" a value. A merged/spanning cell is repeated into each row
  it spans, with `"spanned": true` noted on the table.
- If a table encodes a rule (a per-format limit, a mandatory-field list, a status such as "Already
  Implemented"), also surface it under the model's `rules` / `implementationStatus`, citing the table and
  row, so the spec stage cannot miss it.
- If two tables disagree, record both and their locations.

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
