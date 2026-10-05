---
name: code-generation
description: Write production-oriented source at FileMap targets, matching the reference repo's conventions in the detected language.
---

# code-generation

> Write production-oriented source at FileMap targets, matching the reference repo's conventions in the detected language.

### Procedure
1. Read `FileMap.json`, `Implementation-Spec.json`, `Solution-In-Memory.json`, and the repo profile.
2. Write each source file at its FileMap `targetPath`, in the DETECTED language, FOLLOWING the reference's
   structure, config-resolution, logging and exception CONVENTIONS -- but with content, names and values
   from the solution, not copied from the sibling.
3. Write your report JSON at its EXACT canonical name -- ScaffoldAgent -> `Scaffold.json`, CoreImplementationAgent -> `Core-Implementation.json`, IntegrationImplementationAgent -> `Integration-Implementation.json`, CodeFixAgent -> `Fix-Report.json` (never a `*-report.json` variant) -- listing the files you wrote and any deviations.

### Follow the reference's CONVENTIONS; take the CONTENT from the solution
The reference/sibling teaches you HOW to shape code; the solution design + spec decide WHAT the code is.
Keep the two strictly separate.

STUDY the sibling files named in `Repo-Profile.referenceFiles` only to LEARN its conventions, then
GENERATE fresh code for THIS component from `Solution-In-Memory.json` + the spec, expressed in those
conventions. Conventions to follow: config section style (e.g. grouping config under a section such as `[ssm]` -- the actual section names are READ FROM THE SIBLING AT RUNTIME, never assumed),
array-vs-scalar SHAPE for a kind of value (an ARN list is an array -- e.g. `source-arn`), structured
blocks like `vpc-config` and their sub-key style, key CASING (kebab stays kebab; never rewrite to
camelCase), the pattern of nesting a concern in its own sub-package (an OAuth concern lives in an
`oauth/`-style sub-package), and where tests live relative to the component root.

DO NOT COPY from the sibling: its file CONTENT, its concrete values (real ARNs, URLs, business logic),
or its folder/component/package NAMES. This component's names come from the solution + the run's
component name (never the sibling's name), its files contain only what the solution requires (a concern
the solution does not need gets no module, even if the sibling has one), and config VALUES are the
solution's own (or documented placeholders), merely SHAPED the sibling's way. The alignment check verifies
the code matches the SOLUTION in the reference's conventions -- not that it resembles the sibling.

### Config by convention (placement + section style from the sibling, VALUES from the solution)
Write config where the skeleton says and in the sibling's format: env setup as `<env>_setup.json` at the
OUTER dir, and `.properties` in the INNER `properties/` sub-package using the sibling's EXACT section
headers as found IN THE SIBLING AT RUNTIME (for example `[ssm]`) -- do not assume or hardcode a set of
sections, and never invent one the sibling does not have. The section STYLE and file
placement come from the sibling; the VALUES are the solution's own (or documented placeholders).

### Rules
- Hardcode no language or domain literal that belongs in configuration.
- Docstrings/comments explain WHY: known data quirks, production observations, source-document
  disagreements, intentional design decisions.
- Introduce nothing the spec does not define -- no new field, type, service, persistence or auth mechanism.
- When fixing: repair the implementation to satisfy tests and recorded deviations; NEVER weaken, skip or
  delete a test to make the suite pass.

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
