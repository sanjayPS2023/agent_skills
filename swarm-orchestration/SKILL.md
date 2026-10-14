---
name: swarm-orchestration
description: How an orchestrator delegates to its subagents -- parallel where independent, serial only on a real data dependency, absolute paths handed down, one rollup out.
---

# swarm-orchestration

> How an orchestrator delegates to its subagents -- parallel where independent, serial only on a real data dependency, absolute paths handed down, one rollup out.

You are an orchestrator. You own one stage and a set of focused subagents.

### Delegation
- Give each subagent (1) the RUN ROOT verbatim, (2) its input file paths, and (3) its FULL absolute output
  path at the canonical name (see `workflow-run-contract`). A subagent that has to invent its own
  destination invents the wrong one.
- Run INDEPENDENT subagents in PARALLEL. Run serially only when a subagent reads a file another must
  write first. The real dependencies in this pipeline:
  - 10-analysis: the requirement slices, StandardsChecklist and RepositoryDiscovery run in parallel;
    AnalysisValidation runs last.
  - 20-spec: SolutionInMemory first; then Infrastructure, Integration, BusinessLogic and TestFixture
    designers in parallel; SpecValidation (assembly + verdict) last.
  - 40-codegen: Scaffold first (structure, config, registrations); then Core and Integration
    implementation in parallel; TestAuthoring last (it tests the real public API on disk).
- Never do a subagent's work yourself. Read files with `read_file` -- never spend an agent on that.
- After each subagent returns, check that its file exists, parses, and has `status` OK/PARTIAL. Re-run
  it once with the specific failure reason if not; then record it as failed.

### Your own output
Write ONLY your stage's rollup file (e.g. `Input-Manifest.json`, `Analysis.json`) listing each subagent,
its file and its status. Never write or overwrite a subagent's file, a script-owned file
(`Generated-Files.json`, `Validation-Result.json`, ...) or a stage verdict. Once your rollup exists you
are DONE.

### Resume
If `<RUN ROOT>/_run/resume-plan.json` exists, it lists the `pending` and `done` agents for your stage.
Delegate only the pending ones, and read a done agent's output instead of re-running it. With no plan,
read each expected file yourself and re-run only what is missing or not OK/PARTIAL.

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
