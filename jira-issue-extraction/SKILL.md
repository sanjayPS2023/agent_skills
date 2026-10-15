---
name: jira-issue-extraction
description: Fetch each Jira issue via the Jira MCP tool and capture its description, acceptance criteria (with stable AC ids) and attachments as the definition of done.
---

# jira-issue-extraction

> Fetch each Jira issue via the Jira MCP tool and capture its description,comments , acceptance criteria (with stable AC ids) and attachments as the definition of done.

### Procedure
1. Take the keys from `Run-Inputs.jiraIssueKeys`. For each key, in the order given, call the Jira tool available in this session (e.g. `jira_get_issue`;
   use the real listed name).
2. Capture summary, description, status, labels, attachments (id, filename) and acceptance criteria, all
   VERBATIM.
3. ACs may live in a dedicated field or inside the description (an "Acceptance Criteria" heading, a
   Given/When/Then list). Take them from wherever they are, keep each as a separate item, and record
   `acSource` (`field` or `description`).
4. Number them `<KEY>-AC1`, `<KEY>-AC2`, ... in document order, so later stages can cite them stably.

### Produce
`Jira-Model.json` payload: `issues: [{ "key", "summary", "description", "status", "attachments": [...],
"acceptanceCriteria": [{ "id", "text" }], "acSource" }]`, in input order.

### Failure
- No keys -> `SKIPPED`.
- No Jira tool in the session -> `BLOCKED`, naming it.
- A key that will not fetch -> keep the others, record that key + the error verbatim as a gap, and
  return `PARTIAL`.
- An issue with no ACs anywhere -> record a gap (ACs are the definition of done).

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
