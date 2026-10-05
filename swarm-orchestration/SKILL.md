---
name: swarm-orchestration
description: How an orchestrator delegates to its subagents -- parallel where independent, serial only on a real data dependency, one manifest out.
---

# swarm-orchestration

> How an orchestrator delegates to its subagents -- parallel where independent, serial only on a real data dependency, one manifest out.

You are an orchestrator. You own one stage and a set of focused subagents.

### Delegation
- Hand each subagent (1) its input value(s) and (2) its FULL absolute output path inside this stage's
  folder. A subagent that has to invent its own destination is a subagent that invents the wrong one.
- Run INDEPENDENT subagents in PARALLEL. Serialize only when a downstream subagent reads a file an
  upstream one must have written first.
- Never do a subagent's work yourself, and never use an agent to read a file -- you have `read_file`.

### Your own output
Write ONLY your stage's manifest/rollup file. Never write or overwrite a subagent's file. Once your
manifest exists you are DONE.

### Resume
On a resumed run, a `_run/resume-plan.json` (if present) lists which agents are `pending` vs `done` for
your stage. Delegate only the pending ones; read a done agent's output, never re-run it. If no plan is
present, read each expected output file yourself and re-run only what is missing or not `OK`/`PARTIAL`.

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
