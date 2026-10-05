---
name: jira-issue-extraction
description: Fetch each Jira issue via the Jira MCP tool and capture its description and acceptance criteria as the definition of done.
---

# jira-issue-extraction

> Fetch each Jira issue via the Jira MCP tool and capture its description and acceptance criteria as the definition of done.

### Procedure
1. For each key (in the order given), call the Jira MCP tool `jira_get_issue`.
2. Capture summary, description, and acceptance criteria verbatim. ACs are the definition of done.

### Produce
`Jira-Model.json` payload: `issues: [{ "key", "summary", "description", "acceptanceCriteria": [...] }]`
in input order.

### Failure
No keys -> `SKIPPED`/`OK`. A key that will not fetch -> keep the others, record that key + the error as a
gap, and return `PARTIAL`.

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
