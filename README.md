# agent_skills

Skills loaded by the **ETL Glue code-generation workflow** (`workflow-etl-glue-code-generation`), its repair
loop (`workflow-etl-glue-code-alignment`) and nothing else. Every agent loads `workflow-run-contract` first,
then reads each attached skill's `SKILL.md` from disk before acting.

The workflow creates a NEW Glue service or UPDATES an existing one (in place, or by adding a variant), from a
Confluence solution design, Jira stories, the ETL coding standards `.md`, a reference ETL service and free-text
instructions -- see `workflow-run-contract` for the inputs, file layout and authority order.

## Skill -> agents

| Skill | Used by |
|---|---|
| workflow-run-contract | every agent |
| additional-instruction-triage | AdditionalInstructionAgent |
| alignment-verification | SolutionCaptureVerifyAgent, CodeSolutionAlignmentAgent |
| code-generation | ScaffoldAgent, CoreImplementationAgent, IntegrationImplementationAgent, CodeFixAgent |
| confluence-document-extraction | SolutionModelAgent, SolutionCaptureVerifyAgent |
| gap-review | Gap Presenter, Gap Resolution Agent |
| glue-job-design | TechnicalRequirementAgent, RepositoryDiscoveryAgent, InfrastructureDesignAgent, IntegrationDesignAgent, BusinessLogicDesignAgent, SpecValidationAgent, ScaffoldAgent, CoreImplementationAgent, IntegrationImplementationAgent, TestAuthoringAgent, CodeFixAgent, CodeSolutionAlignmentAgent |
| implementation-spec | InfrastructureDesignAgent, IntegrationDesignAgent, BusinessLogicDesignAgent, SpecValidationAgent, Gap Resolution Agent |
| jira-issue-extraction | JiraModelAgent |
| reference-repo-profiling | ReferenceRepoAgent, RepositoryDiscoveryAgent, ScaffoldAgent, CoreImplementationAgent, IntegrationImplementationAgent |
| requirement-analysis | CodingStandardsAgent, BusinessRequirementAgent, FunctionalRequirementAgent, NonFunctionalRequirementAgent, TechnicalRequirementAgent, DataMappingRequirementAgent, StandardsChecklistAgent |
| solution-document-comprehension | SolutionModelAgent, SolutionCaptureVerifyAgent, SolutionInMemoryAgent |
| solution-in-memory | SolutionInMemoryAgent, Gap Resolution Agent |
| stage-verdict | InputValidationAgent, AnalysisValidationAgent, SpecValidationAgent |
| structured-table-extraction | SolutionModelAgent, CodingStandardsAgent, SolutionCaptureVerifyAgent, DataMappingRequirementAgent |
| swarm-orchestration | Input Ingestion Manager, Requirement Analysis Orchestrator, Implementation Spec Manager, Code Generation Manager |
| test-authoring | TestAuthoringAgent |
| test-fixture-design | TestFixtureAgent |
| validation-and-coverage | TestAuthoringAgent, CodeFixAgent |

## Changelog -- 2026-10-15 (ETL Glue workflow v1)

- NEW `glue-job-design`: Glue runtime/version rules, entrypoint lifecycle, pure transforms vs I/O, job
  arguments/config/secrets, idempotent writes, partitioning, bookmarks, job definition per environment, and
  the mechanical review checklist GLUE-01..12 used by alignment.
- NEW `gap-review`: the Gap Presenter (collect/de-duplicate) and Gap Resolution Agent (apply answers exactly)
  procedures and schemas, which previously lived only in prompts. Answers can be typed (`GAP-001: ...`) or
  uploaded.
- workflow-run-contract: `Run-Inputs.json` (one resolved copy of the form inputs -- reference ETL service,
  service action CREATE/UPDATE/AUTO, target Glue service, staged coding standards); explicit writer per
  canonical file; new files `Validation-Plan.json`, `Baseline-Tests.json`, `Gaps-For-Review.md`,
  `Loop-Stop.json`, rollups `Spec-Manifest.json` / `Codegen-Manifest.json`; the create/extend/update mode
  rule; Glue rules in the authority order; a secrets rule.
- reference-repo-profiling: reference vs TARGET service; `update` mode; mode constrained by the user's
  service action; job-definition conventions; extension points for update mode.
- solution-in-memory: `changeType`, `currentBehaviour`, `preserved` (PRS-*) items, Glue `jobParameters`
  (JOB-*) and `dataContracts` (DATA-*).
- implementation-spec: `update` placement; `job-definition` kind; fixtures merged into the FileMap;
  `Validation-Plan.json` assembly so the validation scripts never guess paths.
- code-generation / test-authoring: update-mode rules (backward compatibility, preserved behaviour, legacy
  tests touched only through a FileMap entry); Glue entrypoint/transform testability; the shared venv.
- validation-and-coverage: matches the new Run Validation script -- one pytest run, regressions vs the
  pre-change baseline, pre-existing failures, tool availability, coverage scope.
- alignment-verification: regression and Glue-runtime deviation kinds; update-mode judging; target name is
  not a "reference name leak".
- stage-verdict: 00-inputs verdict now runs AFTER the solution-capture loop (it used to judge a file not yet
  written); mode/importRoot/Validation-Plan checks.
- swarm-orchestration: real dependency order for the Glue pipeline; resume via `_run/gate-decisions.json`.

## Workflow wiring these skills expect (all done in the Glue workflow)

1. SpecValidationAgent loads `implementation-spec`; TestAuthoringAgent loads `validation-and-coverage`.
2. Run Validation runs on `Validation-Plan.pythonVersion` in `<run dir>/.venv-glue`, records `installExit`,
   `pythonExecutable`, `toolsAvailable`; coverage uses `--include=<generated/modified non-test files>`.
3. The Code Alignment Loop no-ops after `PASS`, or after `Loop-Stop.json` (written when Fix-Report says
   `BLOCKED` with `reason: environment`).
