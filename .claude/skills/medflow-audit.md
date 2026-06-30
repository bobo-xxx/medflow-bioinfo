---
name: "MedFlow Audit"
description: Audit a compiled workflow before execution by verifying configuration, data compatibility, identifiers, node versions, and runtime packages
category: Workflow
tags: [workflow, audit, medflow]
---

Audit a compiled workflow after compilation and again after prerequisite fetch
steps when checks depend on runtime data.

The audit rejects demonstrated incompatibility, not naming differences or
dependencies with a documented, verified installation fallback.

## Severity Model

- **CRITICAL**: proven incompatible data, a missing required parameter, or a
  package that cannot be installed or loaded.
- **WARNING**: a compatible semantic alias, a fallback installation was
  required, or a remote default branch is not named `main`.
- **DEFERRED**: a check requires upstream runtime output that does not exist
  yet. Deferred checks permit only prerequisite fetch steps; repeat the audit
  before merge, DEG, enrichment, or filtering.

## Checks

### 1. Config Key Audit

For every config key on every step:

- Transform underscore to hyphen and add the `--` prefix.
- Match the result against the target node's `parameters[].name` in
  `SKILL.md`.
- A missing required parameter or unsupported config key is **CRITICAL**.
- Report the step, key, transformed parameter, and available parameters.

### 2. Upstream Data Compatibility Audit

For every `bind: upstream` parameter:

- Identify the upstream step and its declared outputs.
- Normalize these semantic aliases before comparison:
  - `gene_expression_matrix`
  - `expression_matrix_gene`
  - `expression_matrix`
  - normalized value: `expression_matrix`
- A normalized match is compatible. Report spelling differences as
  **WARNING**, not failure.
- When files exist, inspect their actual structure:
  - first column contains feature identifiers;
  - remaining expression columns are numeric;
  - sample IDs are compatible with the corresponding metadata or map.
- Mark **CRITICAL** only when structure or content proves incompatibility.
- If required files do not exist yet, mark the structural checks
  **DEFERRED**.

### 3. Group Column Coverage Audit

For group comparisons:

- Open metadata from every upstream dataset.
- Inspect direct columns and keys encoded in `characteristics_ch1`.
- Verify the configured group column or a compiled per-dataset equivalent.
- Count valid group values and warn when any group contains fewer than ten
  samples.
- A missing group with no valid compiled equivalent is **CRITICAL**.
- If fetch metadata does not exist yet, mark this check **DEFERRED**, run only
  the required fetch steps, and repeat the audit before merge or DEG.

### 4. Sample ID Cross-Reference Audit

When the group map and expression matrix exist:

- Verify that column one of the group file contains sample IDs and column two
  contains group labels.
- Cross-reference those IDs against expression-matrix column headers.
- No overlap is **CRITICAL**; partial overlap is a **WARNING** with the matched
  fraction.
- If either file does not exist yet, mark the check **DEFERRED** and repeat it
  after fetch and required inline transformations.

### 5. Node Version Audit

For every node directory:

- Read local HEAD with `git log --oneline -1`.
- Run `git fetch origin`.
- Resolve the remote default branch through `refs/remotes/origin/HEAD`; do not
  assume `origin/main`.
- Compare local HEAD with that remote ref. A stale clone is a **WARNING** and
  must be refreshed before execution.
- A default branch named `master` or anything other than `main` is a
  **WARNING**, not a failure.

### 6. Runtime Package Audit

For every declared environment:

- Parse conda and pip dependencies and check their availability.
- If a package is unavailable from conda on the current platform:
  1. Determine whether it is an R/CRAN or Bioconductor package.
  2. Create the declared isolated environment.
  3. Install CRAN packages with `install.packages()` and Bioconductor packages
     with `BiocManager::install()`.
  4. Verify loading inside that environment with
     `Rscript -e 'library(package)'`.
  5. Report a successful fallback as **WARNING**.
- Distinguish "not available from conda on this platform" from "not
  installable."
- Mark **CRITICAL** only when installation or loading fails.

## Output

Emit one NDJSON result line.

Pass:

```json
{"level":"result","status":"pass","checks":6,"warnings":[],"deferred":[],"critical":[]}
```

Deferred:

```json
{"level":"result","status":"deferred","checks":6,"warnings":[],"deferred":[{"check":3,"step":"fetch1","msg":"Metadata not produced yet; run fetch steps and repeat audit."}],"critical":[]}
```

Failure:

```json
{"level":"result","status":"fail","checks":6,"warnings":[],"deferred":[],"critical":[{"check":1,"step":"fetch1","key":"rename","msg":"Unsupported required configuration."}]}
```

Execution policy:

- `fail`: execute nothing.
- `deferred`: execute only prerequisite fetch steps identified by the audit,
  then repeat the audit.
- `pass`: proceed with the remaining DAG.
