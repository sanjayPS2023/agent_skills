---
name: solution-document-comprehension
description: Read a solution design document and transcribe it into a structured, section-attributed model -- including what already exists vs what is new -- without paraphrase or invention.
---

# solution-document-comprehension

> Read a solution design document and transcribe it into a structured, section-attributed model -- including what already exists vs what is new -- without paraphrase or invention.

The solution design (a Confluence page) is the HIGHEST authority for what gets built. Your job is
transcription and structuring, never design.

### Produce
`Solution-Model.json` payload with at least:
- `sections`: the heading tree with section numbers, so every later citation can point at a section.
- `tables`: every table, via `structured-table-extraction`, values EXACT.
- `architecture`: the business/technical architecture, in structured form.
- `diagrams`: every mermaid/flow diagram, as nodes + edges.
- `rules`: every explicit business rule, limit, contract and edge-case decision, each with its section.
- `implementationStatus`: every statement of what ALREADY EXISTS vs what is NEW ("already implemented",
  "no changes needed", "reuse existing", "new file", "add to ..."), with the exact phrase and section.
  This decides later whether code is reused or written.
- `namedFiles`: every file, module, path or config name the document states, verbatim.
- `referenceComponent`: the existing component/job the document says to extend or follow.
- `attachments`: attachments the document references (id, filename, role).

### Rules
- Never invent content. Record ambiguity as a gap, with its location.
- Where the document disagrees with itself, record BOTH readings and their locations -- do not pick one.
- Keep "already implemented" markers. Dropping them makes later stages re-implement existing logic.

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
