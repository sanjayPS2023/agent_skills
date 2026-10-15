---
name: requirement-analysis
description: Turn captured inputs into one crisp, individually verifiable slice -- a requirement slice, the coding-standards capture, or the standards checklist -- written to its own canonical file.
---

# requirement-analysis

> Turn captured inputs into one crisp, individually verifiable slice -- a requirement slice, the coding-standards capture, or the standards checklist -- written to its own canonical file.

Your prompt says which slice you own. Write ONLY that slice's canonical file -- never a short-named
or extra copy.

| Agent | File |
|---|---|
| BusinessRequirementAgent | `10-analysis/Business-Requirements.json` |
| FunctionalRequirementAgent | `10-analysis/Functional-Requirements.json` |
| NonFunctionalRequirementAgent | `10-analysis/NonFunctional-Requirements.json` |
| TechnicalRequirementAgent | `10-analysis/Technical-Requirements.json` |
| DataMappingRequirementAgent | `10-analysis/Data-Mapping-Requirements.json` |
| CodingStandardsAgent | `00-inputs/Coding-Standards.json` |
| StandardsChecklistAgent | `10-analysis/Standards-Checklist.json` |

### Requirement slices (business / functional / non-functional / technical / data mapping)
Payload: `requirements: [{ "id", "statement", "source", "acceptance", "verifiableIn" }]`.
- `source` cites the solution section, Jira AC or instruction it came from.
- `acceptance` says how it will be checked against generated code or config.
- Technical requirements for a Glue job include the runtime (Glue/Python/Spark version), job
  arguments, sources/targets, formats, partitioning, write mode, scheduling, retries/timeouts and
  worker sizing the inputs state; tag each with `verifiableIn` accordingly.
- `verifiableIn` is one of `code`, `config`, `test`, `infra` (job definition/IaC) or `operational`
  (monitoring, paging or dashboards outside the job). Only `code`/`config`/`test`/`infra` items can be
  checked by the alignment step; `operational` items are recorded but never become code deviations.
- Data mapping additionally keeps `fieldMappings` as rows copied exactly (source field, target field,
  transformation, required).

### Coding standards capture (CodingStandardsAgent)
Read the staged standards document at `Run-Inputs.codingStandardsFile` (the user's ETL Coding Standards
`.md`, copied to `00-inputs/source/ETL-Coding-Standards.md`). If it is `null`, write `SKIPPED` with an
empty `standards` list -- later stages then rely on `glue-job-design` and the reference. Otherwise
transcribe it into
`standards: [{ "id", "category", "rule", "text", "severity": "MANDATORY"|"RECOMMENDED", "section" }]`,
with `text` VERBATIM. Words such as MUST / SHALL / REQUIRED mean MANDATORY; SHOULD / RECOMMENDED mean
RECOMMENDED. Record any numeric threshold the document states (coverage %, line length, timeouts) under
`thresholds` with its section.

### Standards checklist (StandardsChecklistAgent)
Payload:
`{ "coverageMin": <number|null>, "checklist": [{ "id", "category", "rule", "severity", "appliesTo",
"checkItems": [...], "verification" }] }`.
- `coverageMin`: the minimum line-coverage % the standards state, as a NUMBER (e.g. `80`); `null` if
  they state none. The validation script reads this top-level key; without it the default 80 applies.
- `appliesTo`: `new-and-modified` (default) -- checks apply only to files this run creates or edits,
  never to untouched legacy code; or `component` for rules about structure (file placement, env-file
  completeness).
- `verification` is a concrete, mechanical check (a grep, a file existence test, a schema comparison),
  not an opinion, so two runs reach the same result.
- Keep the standards' own ids; one checklist item per rule. Never merge two rules into one item, and
  never let two items demand contradictory things.

### Rules
Derive strictly from the sources; add nothing they do not support. Each item must be individually
verifiable later. Record unclear points as gaps (with the location) rather than resolving them by guess.

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
