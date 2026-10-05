---
name: reference-repo-profiling
description: Locate the reference/sibling component within the cloned repository's sub-packages and profile it to DEDUCE the target language and every convention the generated component must mirror.
---

# reference-repo-profiling

> Locate the reference/sibling component within the cloned repository's sub-packages and profile it to DEDUCE the target language and every convention the generated component must mirror.

The cloned repository is typically a monorepo, and the reference component -- whose name is given in the
solution design (`Solution-Model.json`), not by any UI input -- does NOT necessarily sit at the
repository root: it lives inside the repo's SUB-PACKAGES, often nested several levels deep under
module/service directories. You must first learn its name from the solution model, then FIND its folder,
then profile that folder. That component IS the coding standard; there is no separate standards document,
and the target language/runtime is DEDUCED from the component's own (and governing) files, never taken
from an input or guessed from a name.

### Procedure (read the repo on disk with read_file / command_line)
1. LOCATE the component. Take its name from the solution design (`Solution-Model.json`), then search the
   whole cloned repo for a directory whose name is, or ends with, that name (a sub-package, not assumed
   to be a direct child of the root). If several match, prefer the one that actually contains source +
   tests. Record the resolved absolute path as `componentPath`. If the solution names no reference
   component, or none is found, record that and set `PARTIAL` -- never profile the repo root as a
   fallback.
2. DETECT language/runtime from the manifest/build files that GOVERN that component. These may live IN
   the component folder OR at an ancestor module/repo level (a parent `pom.xml`/`build.gradle`, a
   workspace `package.json`, a shared `requirements.txt`/`pyproject.toml`). Walk UPWARD from
   `componentPath` to the nearest one. (`requirements.txt`/`pyproject.toml`/`setup.py` -> Python;
   `pom.xml`/`build.gradle` -> Java; `package.json` -> Node; etc.) If none is found on the way up, record
   `unknown` and set `PARTIAL`.
3. RECORD, from that component (and its governing module where relevant): package/directory layout,
   naming conventions, configuration-resolution mechanism (e.g. `properties/<env>.properties`), logging
   approach, exception patterns, and test layout + framework.

### Capture the SKELETON as a convention PATTERN (enumerate the tree; placeholders, not names)
Run `ls -R`/`find` on the sibling to LEARN its layout, then record a `skeleton` object -- a PATTERN with a
`<COMPONENT>` PLACEHOLDER, never the sibling's actual name:
- `parentPath`: the real directory components sit in, verbatim (e.g. `src/pace-lambda`). No invented
  prefix -- if the sibling is at `src/pace-lambda/<x>/`, the parent is `src/pace-lambda`, never
  `src/pace-etl-service/pace-lambda`.
- `nesting`: the shape, e.g. `<parent>/<COMPONENT>/<COMPONENT>/` when the sibling double-nests (an OUTER
  dir named for the component, an INNER package of the same name).
- `fileKindToDir`: where each KIND of thing lives, by LEVEL -- env setup -> OUTER `setup/`
  (`<env>_setup.json`); entrypoint (`main.py`), manifest (`requirements.txt`), tests (`test/`), and every
  per-concern sub-package -> INNER; config `.properties` -> INNER `properties/`.
- `subPackageRule`: the RULE that each distinct concern is nested in its OWN sub-package (auth-type code in
  an `auth/` sub-package, utilities in `utilities/`, a domain in a domain-named package) -- record the
  rule, NOT the sibling's specific domain name.
These are PATTERNS to reproduce with the SOLUTION's names, never names/content/concerns to copy.

### Capture EXACT conventions, not a summary
For each concern, record the sibling's REPRESENTATIVE file path AND the exact shapes a generator must copy verbatim -- because a summary loses them and the generator then normalises them away:
- config: the sibling's `properties/<env>.properties` path; its section headers exactly as they appear in the sibling at RUNTIME (for example `[ssm]`; do not assume a fixed set); which values are ARRAYS vs scalars (e.g. `source-arn` is an array); structured blocks (e.g. `vpc-config`) and their sub-keys. Copy key CASING verbatim -- kebab-case stays kebab-case.
- packages: the exact sub-package nesting (e.g. an `oauth/` module lives under the component package), and where the sibling puts its TEST directory relative to the component root.
- entrypoint/handler/logging/exceptions: the sibling file that demonstrates each.
Add these to the payload as `conventions` (section headers, arrayKeys, structuredBlocks, keyCase, package-nesting and test-location PATTERNS) and `referenceFiles` (a map of concern -> absolute sibling file path) so codegen can STUDY the real file to LEARN the convention. These describe HOW code is shaped -- they are patterns to follow, never content or names to copy into the new component.

### Produce
`Reference-Repo-Ref.json` (ingestion) or `Repo-Profile.json` (analysis) payload:
`{ "repo", "branch", "referenceComponent", "componentPath", "governingManifest", "language", "runtime",
"layout", "naming", "configResolution", "logging", "exceptions", "tests": { "framework", "layout" },
"skeleton": { "parentPath", "nesting", "fileKindToDir", "subPackageRule" }, "conventions", "referenceFiles" }`.
`componentPath` is the resolved sub-package path; every field cites where in the component it was found.

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
