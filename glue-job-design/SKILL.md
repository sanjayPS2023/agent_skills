---
name: glue-job-design
description: AWS Glue platform rules for designing, generating, testing and reviewing an ETL Glue job -- entrypoint lifecycle, pure transforms vs I/O, job arguments and config, DynamicFrame/DataFrame use, bookmarks and idempotent writes, job definition keys, and a mechanical review checklist.
---

# glue-job-design

> AWS Glue platform rules for designing, generating, testing and reviewing an ETL Glue job -- entrypoint lifecycle, pure transforms vs I/O, job arguments and config, DynamicFrame/DataFrame use, bookmarks and idempotent writes, job definition keys, and a mechanical review checklist.

These rules make a Glue job CORRECT on the Glue runtime. They sit below the ETL Coding Standards and
above the reference's conventions (see the authority order in `workflow-run-contract`). Where the
reference service already solves a concern a particular way (its own config loader, logger, stub), follow
the reference; these rules then only constrain what the reference leaves open.

### 1. Runtime and versions
- Target `Repo-Profile.glueVersion` / `pythonVersion` / `sparkVersion`. Glue 5.0 = Python 3.11 / Spark
  3.5; Glue 4.0 = Python 3.10 / Spark 3.3; Glue 3.0 = Python 3.7 / Spark 3.1. Use no syntax or API newer
  than that Python and Spark.
- `jobType`: `glueetl` (Spark), `pythonshell` (no Spark, no GlueContext), `gluestreaming`. Never mix:
  a Python-shell job must not import pyspark or awsglue.
- Extra libraries: only those in the governing manifest or the job's `--additional-python-modules` /
  `--extra-py-files`. A new dependency is a design decision recorded in Infrastructure-Design, never a
  silent import.

### 2. Entrypoint shape (glueetl)
```
args = getResolvedOptions(sys.argv, [JOB_NAME, <declared args>])   # only declared, required args
sc = SparkContext.getOrCreate(); glue_context = GlueContext(sc); spark = glue_context.spark_session
job = Job(glue_context); job.init(args["JOB_NAME"], args)
config = load_config(args, ...)          # the reference's config resolution (env, properties, setup)
run(spark/glue_context, config)          # read -> transform -> write, in separate functions
job.commit()                             # only after every write succeeded
```
- The module must be importable without side effects: put the lifecycle under
  `if __name__ == "__main__":` (or the reference's `main()` convention). Tests import it.
- Optional arguments: `getResolvedOptions` fails on a missing arg, so optional ones are read with a guard
  (`"--x" in sys.argv`) or declared with defaults in the job definition -- follow the reference.
- Fail loudly: let exceptions propagate after logging (a swallowed exception marks a failed load as
  SUCCEEDED). Never call `job.commit()` in a `finally`.

### 3. Separate pure transforms from I/O
- Readers (catalog / S3 / JDBC / connections), transforms, and writers live in separate functions or
  modules per the reference layout. Transforms take and return DataFrames (or plain records) and never
  touch `glue_context`, S3 paths or boto3 -- that is what makes them unit-testable offline.
- Convert DynamicFrame <-> DataFrame at the I/O boundary (`toDF()`, `DynamicFrame.fromDF(...)`) unless the
  reference keeps DynamicFrames through transforms (e.g. for `ApplyMapping`, `ResolveChoice`).
- Field mappings from the solution (`MAP-*`) are applied in ONE place (an `ApplyMapping` mapping list or
  a select/cast/alias list built from config/constants), so alignment can check them row by row.

### 4. Config, arguments and secrets
- Environment-specific values (buckets, prefixes, database/table names, connection names, thresholds)
  come from job arguments or the reference's per-env config/properties files -- one per environment in
  `Repo-Profile.envFiles`. Never hardcode an account id, ARN, bucket or credential in code.
- Credentials: Glue connections or Secrets Manager (by secret NAME from config). Never in code, config,
  tests or fixtures.
- Constants (column names, enum values, formats) live in the service's constants module.

### 5. Data correctness
- Schema: declare the expected input schema (StructType or a mapping list) for each source the solution
  defines; validate required columns at read time and fail with a clear message when absent.
- Nulls, types and dedup: implement exactly the solution's rules (`BR-*`, `EDGE-*`); never drop rows
  silently -- route rejected records to the reference's error/quarantine path or count and log them.
- Writes must be idempotent for a re-run of the same input: partition overwrite
  (`spark.sql.sources.partitionOverwriteMode=dynamic`), a staging path + swap, or job bookmarks -- choose
  what the solution/reference uses and record it as a `DEC-*`. Never plain `append` into the final
  location without a dedup key.
- Partitioning: partition keys, file format (parquet/orc/csv/json), compression and output path layout
  exactly as the solution states (`DATA-*`). Push down predicates (`push_down_predicate`) on partitioned
  catalog sources when the solution filters by partition.
- Job bookmarks: when enabled, every source read passes a stable `transformation_ctx`, and
  `job.init`/`job.commit` bracket the run.
- Avoid driver-side collection of datasets (`collect()`, `toPandas()`) except for bounded lookups the
  solution sizes.

### 6. Logging and observability
- Use the reference's logger (or `glue_context.get_logger()`); no `print`. Log job start/end with the
  run's key arguments (never secrets), row counts read/written/rejected per step, and the error before
  re-raising.
- Metrics/alerts the solution requires are `infra`/`operational` items: the job definition or the
  reference's metrics helper -- not ad-hoc code.

### 7. Job definition (Infrastructure-Design / setup files)
For every environment record and generate, in the reference's job-definition format: job name pattern,
`GlueVersion`, `WorkerType`, `NumberOfWorkers`, `Timeout`, `MaxRetries`, `MaxConcurrentRuns`, script
location, `--job-bookmark-option`, `--enable-metrics`, `--enable-continuous-cloudwatch-log`,
`--extra-py-files` / `--additional-python-modules`, connections, default arguments the entrypoint
declares, and the trigger/schedule if the solution defines one. Values come from the solution or the
reference's same-environment setting; anything unknown is a gap with a proposed default, not an invented
value. Every argument the entrypoint reads with `getResolvedOptions` must be defined in every
environment's job definition.

### 8. Testability
- Tests run offline on the target Python without a Glue runtime: stub or patch `awsglue`; use a local
  SparkSession only where DataFrame semantics are needed (see `test-authoring`).
- Keep functions small enough that each `BR-*`/`EDGE-*` has one obvious unit to test.

### Review checklist (alignment uses these ids; each is mechanical)
| Id | Check |
|---|---|
| GLUE-01 | Entrypoint lifecycle present: `getResolvedOptions` -> `job.init` -> ... -> `job.commit` (glueetl), and no `commit` inside `finally`/`except`. |
| GLUE-02 | Entrypoint module has no import-time side effects (lifecycle under `__main__`/`main()`). |
| GLUE-03 | Transform modules import neither `awsglue` nor `boto3` (`grep -n "awsglue\|boto3" <transform files>`). |
| GLUE-04 | Every `getResolvedOptions` key appears in every environment's job definition / setup file. |
| GLUE-05 | No hardcoded account id (`\b\d{12}\b`), `arn:aws:`, `s3://` literal or credential in generated non-test code; such values come from args/config. |
| GLUE-06 | Each `MAP-*` field mapping is present once, with the solution's source/target names and type. |
| GLUE-07 | Writes use the recorded idempotency approach (`DEC-*`); no bare `mode("append")` into a final path without a dedup key. |
| GLUE-08 | Partition keys, format and compression match `DATA-*`. |
| GLUE-09 | Bookmarked reads carry a `transformation_ctx`. |
| GLUE-10 | No `print(`; row counts logged per read/write. |
| GLUE-11 | Job definition per environment has GlueVersion = `Repo-Profile.glueVersion` and the solution's worker type/count, timeout and retries. |
| GLUE-12 | No `collect()`/`toPandas()` on an unbounded dataset. |
Failures of GLUE-01..06 and GLUE-11 are `blocking`; the rest are `blocking` only when the solution or the
standards state the rule, else `warning`.

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
