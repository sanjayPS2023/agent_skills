---
name: solution-document-comprehension
description: Read a solution design document and transcribe it into a structured model without paraphrase or invention.
---

# solution-document-comprehension

> Read a solution design document and transcribe it into a structured model without paraphrase or invention.

The solution design (a Confluence page) is the HIGHEST authority for what gets built. Your job is
transcription and structuring, never design.

### Produce
`Solution-Model.json` payload with at least:
- `tables`: every table, values EXACT (a CIDR, ARN, action name, limit or format is copied, not summarised).
- `architecture`: the business/technical architecture described, in structured form.
- `diagrams`: any mermaid/flow diagram, interpreted into nodes + edges.
- `rules`: every explicit business rule, limit, contract and edge-case decision, each attributed to the
  section it came from.

### Rules
- Never invent content the document does not contain. If something is ambiguous, record it as a gap.
- Where the document disagrees with itself, record BOTH readings and the location -- do not silently pick one.

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
