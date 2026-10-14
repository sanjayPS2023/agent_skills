# agent_skills

Skills loaded by the ETL Glue code-generation workflow (`workflow-etl-glue-code-generation-v2-gates-fixed`)
and its repair loop (`workflow-etl-glue-code-alignment-FIXED`). Every agent loads `workflow-run-contract`
first, then reads each skill's `SKILL.md` from disk before acting.

## Skill -> agents

| Skill | Used by |
|---|---|
| workflow-run-contract | every agent |
| swarm-orchestration | Input Ingestion Manager, Requirement Analysis Orchestrator, Implementation Spec Manager, Code Generation Manager |
| confluence-document-extraction | SolutionModelAgent, SolutionCaptureVerifyAgent |
| solution-document-comprehension | SolutionModelAgent, SolutionCaptureVerifyAgent, SolutionInMemoryAgent |
| structured-table-extraction | SolutionModelAgent, CodingStandardsAgent, DataMappingRequirementAgent |
| jira-issue-extraction | JiraModelAgent |
| additional-instruction-triage | AdditionalInstructionAgent |
| reference-repo-profiling | ReferenceRepoAgent, RepositoryDiscoveryAgent, Scaffold/Core/IntegrationImplementationAgent |
| requirement-analysis | Business/Functional/NonFunctional/Technical/DataMapping requirement agents, CodingStandardsAgent, StandardsChecklistAgent |
| solution-in-memory | SolutionInMemoryAgent |
| implementation-spec | Infrastructure/Integration/BusinessLogic design agents, **SpecValidationAgent (add -- see below)** |
| test-fixture-design | TestFixtureAgent |
| code-generation | Scaffold/Core/IntegrationImplementationAgent, CodeFixAgent |
| test-authoring | TestAuthoringAgent |
| validation-and-coverage | CodeFixAgent, **TestAuthoringAgent (add)** |
| alignment-verification | SolutionCaptureVerifyAgent, CodeSolutionAlignmentAgent |
| stage-verdict | InputValidationAgent, AnalysisValidationAgent, SpecValidationAgent |

## Workflow wiring changes these skills expect

1. **SpecValidationAgent** must also load `implementation-spec`. The FileMap/Implementation-Spec assembly
   rules live there, and today the assembler only loads `stage-verdict`.
2. **TestAuthoringAgent** should also load `validation-and-coverage` (for the environment-vs-code triage
   and the coverage scope).
3. **Run Validation** must run on `Repo-Profile.pythonVersion` (Glue 4.0 -> Python 3.10) in a venv, record
   `installExit` and `pythonExecutable` in `Validation-Result.json`, set `BLOCKED` when the install fails,
   and measure coverage with `--include=<generated non-test files>`.
4. **Gate / loop**: stop the Code Alignment Loop on `PASS`, or when Fix-Report says `BLOCKED` with
   `reason: environment`.

## Changelog -- 2026-10-14 (from the r5 DCSNet run)

- workflow-run-contract: RUN ROOT = the `workflow_output` folder itself (the old text said it *contains*
  `workflow_output/`, the exact confusion that sent agents to wrong paths); added `30-gaps`, the full
  canonical file table with owners (`Coding-Standards`, `Data-Mapping-Requirements`, `Standards-Checklist`,
  `Static-Checks`, gap files); removed the never-written `Toolchain-Profile.json`; added the authority
  order, absolute-path discipline and the source-code write boundary.
- reference-repo-profiling: removed "there is no separate standards document" (it contradicted the ETL
  Coding Standards input); added create/extend `mode`, top-level `glueVersion`/`pythonVersion`/
  `sparkVersion` (the validation script reads `glueVersion` at the top level; r5 had it nested), the
  test command and import root, the complete `envFiles`, and `extensionPoints`; replaced the Lambda examples.
- requirement-analysis: now covers the Data-Mapping, Coding-Standards and Standards-Checklist outputs
  (previously "write only one of the four requirement files", which contradicted those agents);
  `coverageMin` is required (r5 had `null`); `verifiableIn`; standards apply only to new/modified files.
- solution-in-memory: stable ids plus `realisation` (reuse / new-code / config / infra / operational) with
  `reuseTarget`; gap resolutions applied. Fixes r5's DEV-001..010/026..032, which demanded
  re-implementation of logic the solution said to reuse.
- implementation-spec: FileMap entries now carry `kind`, `action` (create/modify), `implements` and
  `reuses`; per-env, registration, config and fixture entries are mandatory; extend mode is supported;
  the assembly rules are explicit. (r5's FileMap had 2 entries, so setup/properties/init/fixture files
  were "missing" on every iteration.)
- code-generation: reuse-by-delegation; modify-in-place for registrations; target Python version; raw-string
  regexes (r5 static.log warnings); no manifest edits; verify with py_compile plus pytest; repair mode
  triages the environment first, reads the previous Fix-Report, and never undoes a repair without a
  higher authority (r5 created and then deleted `DCSNetProcessRule.py`, and added and then removed
  `cache.invalidate`).
- test-authoring: offline and mocked, on the runtime Python; creates every fixture it opens (r5 DEV-024/034);
  asserts each SIM id; wiring tests for reused logic; coverage measured on generated files; runs the tests itself.
- validation-and-coverage (was missing from this repo): full triage table (install failure, wrong
  interpreter, pytest exit 2 = zero tests ran), environment vs code vs pre-existing classification,
  coverage scope, re-verification, and a `BLOCKED: environment` exit for the loop.
- alignment-verification: output schema now matches the workflow prompt; reuse-aware; disk evidence
  required for "missing" (r5 DEV-013 flagged a file that existed); one deviation per rule and file with
  no contradictions (r5 DEV-015 vs DEV-016); stable ids, carrying forward the previous result;
  `operational` items never block.
- stage-verdict: flat file (no envelope) made explicit; stage-specific checks added.
- test-fixture-design (was missing from this repo): concrete paths, built from authoritative samples.
- swarm-orchestration, additional-instruction-triage, confluence, jira, solution-document-comprehension,
  structured-table-extraction: real dependency order; tool names not hardcoded; attachments; stable AC
  ids; "already implemented" markers preserved.
