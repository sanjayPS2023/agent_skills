---
name: confluence-document-extraction
description: Fetch a Confluence page by id or URL via the Confluence MCP tool and hand its full content to the comprehension step.
---

# confluence-document-extraction

> Fetch a Confluence page by id or URL via the Confluence MCP tool and hand its full content to the comprehension step.

### Procedure
1. Resolve the page id: if given a URL, the id is the numeric segment after `/pages/`.
2. Call the Confluence MCP tool `confluence_get_page` with that id, `convert_to_markdown=true`,
   `include_metadata=true`. Actually retrieve -- reporting OK for a page you did not fetch is forbidden.
3. Pass the full page (tables as tables, not flattened) to solution-document-comprehension.

### Failure
If no page was supplied -> `SKIPPED`/`OK`, no content. If the page cannot be retrieved -> `PARTIAL`
with the page identifier and the error text verbatim. Never invent page content.

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
