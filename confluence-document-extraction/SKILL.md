---
name: confluence-document-extraction
description: Fetch a Confluence page (and its referenced attachments) by id or URL via the Confluence MCP tool and hand its full content to the comprehension step.
---

# confluence-document-extraction

> Fetch a Confluence page (and its referenced attachments) by id or URL via the Confluence MCP tool and hand its full content to the comprehension step.

### Procedure
1. Resolve the page id. If you are given a URL, the id is the numeric segment after `/pages/`.
2. Fetch the page with the Confluence MCP tool available in this session (e.g. `confluence_get_page`,
   with markdown conversion and metadata ON). Use the tool's real name as listed; do not assume one.
   Actually retrieve the page -- reporting OK for a page you did not fetch is forbidden.
3. List the page's attachments (id, filename, media type). Fetch the ones the page or a gap resolution
   names as authoritative (schemas, sample feeds, mapping sheets) when the tool supports it, and save
   them under `<RUN ROOT>/00-inputs/source/`; otherwise record their ids so later stages can cite them.
4. Pass the full page -- headings with their section numbers, tables as tables (not flattened), code
   blocks and diagrams intact -- to `solution-document-comprehension`.

### Failure
- No page supplied -> `SKIPPED`, no content.
- No Confluence tool in the session -> `BLOCKED`, naming the missing tool.
- Fetch error -> `PARTIAL`, with the page id and the error text verbatim.
Never invent page content.

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
