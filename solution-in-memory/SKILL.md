---
name: solution-in-memory
description: Build the canonical, id-stable solution-in-memory the generated code is checked against -- every rule tagged with HOW it is realised (reuse, new code, config, infra, operational).
---

# solution-in-memory

> Build the canonical, id-stable solution-in-memory the generated code is checked against -- every rule tagged with HOW it is realised (reuse, new code, config, infra, operational).

This is the single most important design artifact. Alignment diffs the code against it, and the repair
agent repairs toward it. If an item is ambiguous here, those two agents will disagree with each other on
every iteration.

### Inputs (in authority order)
`Solution-Model.json` + `Jira-Model.json` ACs, amended by `30-gaps/Gap-Resolutions.json` when it exists
(a resolution REPLACES the default it resolves -- cite the gap id) and narrowed by
`Additional-Instruction.json`. Use `Repo-Profile.json` only to resolve "reuse X" to a real module.

### Produce
`Solution-In-Memory.json` payload with these sections:
- `businessRules` (`BR-*`), `contracts` (`CT-*`), `limits` (`LIM-*`), `formats` (`FMT-*`),
  `decisions` (`DEC-*`), `edgeCases` (`EDGE-*`), `fieldMappings` (`MAP-*`).
- `implementationFiles`: `{ "create": [...], "modify": [...], "unchanged": [...] }` with the explicit
  paths or names the solution or Jira state. These have precedence (1) when later stages resolve paths.

Every item has:
```json
{ "id": "BR-PRICE-MIN", "statement": "...", "condition": "...", "outcome": "...",
  "values": { "minimumPrice": 999 }, "source": "<section / AC / gap id>",
  "realisation": "reuse | new-code | config | infra | operational",
  "reuseTarget": "<module.function in the repo, when realisation=reuse>",
  "verifiableIn": "code | config | test | infra | none" }
```

### Realisation (decide it once, here)
- `reuse`: the solution says the behaviour already exists ("already implemented", "use existing rules",
  "follow the <X> codebase"). Resolve `reuseTarget` to the real module/function (grep the repo). The new
  code must DELEGATE to it; it must not re-implement it. Alignment checks that delegation, not a copy.
- `new-code`: genuinely new behaviour the new variant must implement.
- `config`: satisfied by a value in a config/properties file (a threshold, flag, path or market entry).
- `infra`: satisfied by the job definition / setup files (worker type, timeout, retries, connections).
- `operational`: monitoring, paging, dashboards or cross-system checks outside the job (e.g. "page the
  on-call team", "reflected consistently on the front-end"). Record these with `verifiableIn: none`;
  they never become code deviations.

### Rules
- Introduce nothing not supported by the inputs. Phrase each item so it can be checked mechanically, with
  exact values in `values`.
- Ids are STABLE: derive them from the rule's meaning (`BR-DUPLICATE-VIN`), never from list position, and
  keep the same id on a re-run so iterations stay comparable.
- Preserve documented disagreements as a `DEC-*` that records both readings, which one was chosen, and why.
- Never tag an item `new-code` when the solution says it already exists. That contradiction is what made
  earlier repair loops create a module in one pass and delete it in the next.

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
