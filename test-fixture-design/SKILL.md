---
name: test-fixture-design
description: Design reusable test fixtures from the solution's real edge cases and sample files -- each with a concrete file path, real structure and derived (not magic) expectations.
---

# test-fixture-design

> Design reusable test fixtures from the solution's real edge cases and sample files -- each with a concrete file path, real structure and derived (not magic) expectations.

### Inputs
`Solution-In-Memory.json` (edge cases, limits, formats, field mappings), `Repo-Profile.json`
(`tests.resourcesDir`, the existing fixture style), and any sample or schema file the solution or a gap
resolution names as authoritative (e.g. an attached feed or XSD).

### Produce
`Test-Fixtures.json` payload:
```json
{ "fixtures": [{ "name": "valid_feed_minimal", "purpose": "...", "covers": ["BR-...", "EDGE-..."],
                 "targetPath": "<ABSOLUTE path under tests.resourcesDir, or null for inline data>",
                 "format": "xml|json|csv|inline", "derivedFrom": "<sample file / table / SIM id>",
                 "content": "<the content, or a precise construction recipe>",
                 "expected": { "...": "value, or 'derive via <helper>'" } }],
  "uncoveredIds": [] }
```

### Rules
- Cover every `new-code` / `config` edge case and every limit boundary (below, at and above the limit),
  plus one valid end-to-end sample and one malformed sample per supported format.
- When an authoritative sample/schema exists, build fixtures from its real element and field names --
  never an invented shape.
- Every fixture a test will open has a concrete `targetPath`; the spec adds it to the FileMap as a
  `test-resource` and test authoring creates it. A fixture that is referenced but never created is a
  standards failure.
- Where an expected value depends on a reusable pure helper, write "derive via <helper>" rather than a
  literal that can drift from the implementation.
- For Glue transforms, prefer small row lists (JSON lines / CSV) that a test turns into a DataFrame with
  the declared schema, plus one fixture per source schema variant the solution lists.
- Keep fixtures small and anonymised (fake ids, no real customer data, no credentials or real buckets).

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
