---
name: "MedFlow Audit"
description: Audit and, when safe, remediate a compiled workflow by verifying configuration, data compatibility, identifiers, pinned node revisions, and runtime packages; produce labeled reproducible code for every repair
category: Workflow
tags: [workflow, audit, medflow]
---

Audit a schema-2.0 compiled workflow after compilation, its initial full plan
snapshot before execution, and proposed full plan snapshots created for reruns,
replacement nodes, or downstream replanning. Schema `1.0` and `1.1` workflows
are not executable and must be recompiled.

In paths below, `<workflow>` means the immutable compiled workflow `name`
field, validated as a safe single directory component.
At audit start, generate one collision-resistant `audit_id`; use it for every
audit clone, remediation-authoring clone, log, and registry event in that
invocation.

The audit rejects demonstrated incompatibility, not naming differences or
dependencies with a documented, verified installation fallback.

The audit may edit a fresh remediation-authoring clone of a pinned node or
write adapter/helper code when a finding can be repaired without silently
changing scientific meaning. Every
repair must be labeled, reproducible from the pinned inputs and node commit,
tested, and re-audited before execution continues.

## Audit Modes

- **INITIAL_PLAN**: verify the complete first plan snapshot against the
  immutable compiled workflow before unrestricted dispatch.
- **RERUN_PARAMETERS**: compare proposed effective parameters with the original
  protocol, compiled workflow, preceding relevant attempt, pinned node
  contract, upstream semantics, and downstream compatibility.
- **NODE_REPLACEMENT**: verify that a registry replacement satisfies the
  original step intent, method suitability, semantic inputs, required result
  semantics, parameter mapping, and environment requirements.
- **DOWNSTREAM_REPLAN**: trace a changed upstream selection or replacement
  through every dependent edge and verify the complete proposed downstream
  subgraph before its full plan snapshot becomes active.
- **RUNTIME_FINDING**: evaluate a terminal attempt review or interrupted
  attempt before the run agent records its disposition.

When `workflow-run.json` exists, verify that a candidate plan's `based_on`
value names the current `active_plan_id`. Audit the complete candidate
snapshot; never infer an effective plan by replaying deltas.

## Severity Model

- **CRITICAL**: proven incompatible data, a missing required parameter, or a
  package that cannot be installed or loaded.
- **WARNING**: a compatible semantic alias, a fallback installation was
  required, or a remote default branch is not named `main`.
- **DEFERRED**: a check requires upstream runtime output that does not exist
  yet. Deferred checks permit only prerequisite fetch steps; repeat the audit
  before merge, DEG, enrichment, or filtering.

## Remediation Authority

Diagnose and assign a stable finding ID before changing anything. The audit may
then perform one of these remediations:

- **INPUT_ADAPTER**: write deterministic code that creates a new run-local
  derived input, such as column renaming, exact label filtering, transposition,
  feature subsetting, or schema normalization.
- **NODE_PATCH**: edit a fresh remediation-authoring clone of the pinned node to
  fix an implementation defect while preserving the declared scientific method
  and public contract.
- **HELPER_CODE**: write a missing deterministic conversion, validation, or
  orchestration helper required to connect declared-compatible artifacts.
- **ENVIRONMENT_FIX**: amend only the isolated runtime environment or install a
  documented dependency fallback.

The audit must not automatically make a change that alters the biological
contrast, statistical method, normalization, identifier semantics, imputation,
feature-selection meaning, model target, output schema, or public node
contract. Such a change requires explicit user confirmation and corresponding
protocol, registry, and node `SKILL.md` documentation updates.

Never overwrite or modify raw/source input files. Write derived inputs beneath:

```text
<run-root>/audit-remediation/<candidate-plan-id>/<finding-id>/outputs/
```

Do not push, commit, or publish a node patch unless the user separately asks.

## Affected Input Inventory

Before writing an `INPUT_ADAPTER`, report each affected file with:

- absolute run-local path and SHA-256;
- producing step or external-input provenance;
- observed format, delimiter, dimensions, identifier column, required columns,
  and a small non-sensitive schema sample;
- the exact incompatibility and downstream parameter it blocks;
- whether the proposed change is mechanical or changes scientific meaning;
- adapter code path, derived output path, expected schema, and acceptance
  checks.

Label the source file `immutable: true`. The adapter must read it and create a
new finding-ID-labeled output; it must never edit the file in place.

## Labeled Reproducibility Bundle

Before a workflow run exists, store an initial-audit repair beneath:

```text
workflows/remediations/<workflow>/<finding-id>/
├── remediation.yaml
├── code/
├── node.patch                 # NODE_PATCH only
├── run_command.ps1            # Windows, when applicable
├── run_command.sh             # POSIX, when applicable
├── environment.yaml           # or an equivalent lock/session record
├── checksums.sha256
├── tests/
└── logs/
```

During a workflow run, store the bundle beneath the current run instead:

```text
<run-root>/audit-remediation/<candidate-plan-id>/<finding-id>/bundle/
```

Never share or overwrite a remediation bundle across workflow runs or candidate
plans. The runtime bundle and its derived outputs use the same candidate-plan
and finding identity.

`remediation.yaml` must label the repair with:

- `label: MEDFLOW_AUDIT_GENERATED_REMEDIATION`;
- finding ID, remediation type, UTC creation time, and rationale;
- workflow path and pre-remediation workflow SHA-256;
- node name, registry URL, registry/contract version, release tag/commit,
  default branch, and pinned base commit when a node is involved;
- raw input paths and SHA-256 checksums without copying or changing them;
- every generated, modified, and derived file;
- exact command, working directory, environment, random seeds, and expected
  outputs;
- whether scientific meaning or the public contract changed;
- tests run, exit codes, observed outputs, and post-remediation audit result.

Begin generated source files with a language-appropriate comment containing
`MEDFLOW_AUDIT_GENERATED_REMEDIATION`, the finding ID, purpose, and bundle path.
For formats that cannot contain comments, such as CSV, use a filename containing
the finding ID and record the label in a sidecar `remediation.yaml` entry.

For a node edit, preserve the registry commit as the immutable base, export the
complete change as `node.patch`, and record its SHA-256. A future run must clone
the pinned base commit and apply that exact patch; an unrecorded dirty checkout
is never an executable source.

To author a node patch, generate a unique `authoring_id`, clone the registry URL
fresh into:

```text
runs/<workflow>/audit/<audit-id>/remediation-authoring/<finding-id>/<authoring-id>/node
```

Checkout the pinned base commit detached, verify release/contract identity,
make the edit there, and export the complete diff into the remediation bundle.
Record the authoring workspace, clone command, base/diff hashes, and disposition
in the audit registry. Preserve the authoring workspace as evidence; never use
a shared `nodes/` checkout or execute its dirty tree directly.

Use this minimum manifest shape:

```yaml
schema: medflow-audit-remediation@1
label: MEDFLOW_AUDIT_GENERATED_REMEDIATION
finding_id: MF-AUD-001
type: INPUT_ADAPTER  # INPUT_ADAPTER | NODE_PATCH | HELPER_CODE | ENVIRONMENT_FIX
created_at_utc: "<ISO-8601>"
scientific_meaning_changed: false
public_contract_changed: false
workflow:
  path: workflows/<name>.json
  sha256: "<sha256>"
node:
  name: "<node-or-null>"
  url: "<registry-url-or-null>"
  version: "<version-or-null>"
  contract_version: "<version-or-null>"
  release_tag: "<tag-or-null>"
  release_commit: "<sha-or-null>"
  base_commit: "<sha-or-null>"
inputs:
  - path: "<immutable-source-path>"
    sha256: "<sha256>"
    immutable: true
artifacts:
  - path: code/<script>
    sha256: "<sha256>"
    label: MEDFLOW_AUDIT_GENERATED_REMEDIATION
reproduction:
  working_directory: "<sandbox-relative-path>"
  command: "<exact-command>"
  environment: environment.yaml
  seeds: {}
verification:
  tests: []
  smoke_test_exit_code: null
  reaudit_status: null
```

## Remediation Verification

After writing a repair:

1. Recreate or verify the repair from its recorded command in an isolated
   remediation directory.
2. Verify all input, code, patch, environment, and output checksums.
3. Run focused unit/validation tests and a minimal affected-step smoke test when
   safe test data are available.
4. Re-run every audit check affected by the finding.
5. For a node patch, start from a clean clone of the pinned commit, apply
   `node.patch`, and prove the resulting diff and code hashes match the bundle.
6. For initial audit before a workflow run exists, write
   `workflows/<name>.audited.json` containing the original compiled workflow
   SHA-256 plus references and checksums for all accepted remediation bundles.
   For runtime audit, write those references into the complete run-local
   candidate plan snapshot instead. Do not overwrite the original compiled
   workflow or bypass `active_plan_id`.

Only a successfully reproduced and re-audited repair may change a finding from
CRITICAL to remediated.

## Runtime Agent-Review Findings

`medflow-run` performs an automatic agent review after every node attempt. When
that review returns `rerun_recommended` or `replacement_recommended` because
adapter/helper code or a node edit is needed, treat the workflow-level
`<run-root>/reviews/<attempt-uuid>.json` as a new audit finding and apply the
remediation rules above. Preserve the terminal attempt workspace unchanged;
include its attempt UUID, path, recorded hashes, and review path/hash in
`remediation.yaml` as the triggering evidence.

After remediation, re-audit the affected node, its input/output contract, and
all downstream assumptions invalidated by the change. Return an updated
full plan snapshot with the remediation bundle reference. The run agent then
reruns the node in a new UUIDv4 workspace and reviews it again. Routine
agent review and contract-preserving retries do not require user approval;
scientific-meaning or public-contract changes still do.

The attempt under review is already terminal and immutable. Audit must never
repair it in place. Any correction runs in a new attempt workspace. A
replacement or downstream node/edge change must be materialized as a new
complete plan snapshot before dispatch.

## Checks

### 0. Workflow and Attempt State Audit

- Require the compiled workflow and every plan snapshot to use schema `2.0`.
- For a new run, verify the initial full plan equals the compiled workflow's
  executable content and records its path and SHA-256.
- For a resumed run, load `workflow-run.json.active_plan_id` and reject any
  instruction that dispatches directly from the original `workflow.json`.
- Verify plan IDs and attempt IDs are UUIDv4 values. Verify workflow-run IDs use
  a UTC timestamp plus random suffix; chronology comes from timestamps.
- Verify workflow and attempt locks. Audit may inspect an active run but must
  not mark it terminal, clear locks, or authorize cleanup.
- Verify every consumed upstream workspace is terminal, selected, not stale,
  and named explicitly in the candidate bindings. Its recorded plan ID, node
  source, effective parameters, and bindings must match the candidate active
  plan.
- Verify every terminal workspace remains immutable.

### 0a. Parameter Rerun Audit

For `RERUN_PARAMETERS`, record complete old and proposed effective parameter
maps and a structured difference. Approve autonomous change only when every
value is accepted by the pinned node contract, the original scientific intent
remains unchanged, input and result semantics remain compatible, and affected
dependencies are handled in the candidate plan. Require every changed argument,
path, config value, parameter, or file binding to appear in a new complete
candidate plan snapshot; reject registry-only or adjustment-only overrides.

Apply the authoritative meaning-changing predicate below.

### 0b. Replacement and Cascading Dependency Audit

For `NODE_REPLACEMENT` or `DOWNSTREAM_REPLAN`, compose all applicable audit
modes in one audit of the same complete candidate plan. A replacement that also
changes parameters or downstream structure therefore includes
`RERUN_PARAMETERS` and/or `DOWNSTREAM_REPLAN` checks.

1. Resolve the replacement only from `registry.yaml`, clone it fresh, and pin
   its repository, version, default branch, commit, and contract hash.
2. Compare it with the immutable step `intent`: capability, scientific goal,
   required semantic inputs, and required semantic outputs.
3. Verify method suitability, parameter mapping, environment viability, output
   equivalence or an explicit reproducible adapter, and every outgoing edge.
4. Require every revised-plan activation to atomically clear selections for
   each changed step and its affected downstream subgraph, then mark prior
   attempts stale while preserving their workspaces unchanged.
5. Audit each affected downstream action: rebind, rerun, replace, halt, or
   escalate.
6. Permit an independent branch to continue only when it has no data,
   quality-gate, or scientific dependency on the changed path.
7. Require a new complete plan snapshot with UUID `plan_id` and `based_on`.
   Reject delta-only execution instructions.

Apply the authoritative meaning-changing predicate below.

### 0c. Meaning-Changing Change Predicate

Across every audit mode, a change is meaning-changing when it alters
specifications, approved scientific design or declared workflow intent,
scientific intent or objectives, public contracts, required result semantics,
protected assumptions, or downstream interpretation. A contract-compatible
implementation substitution or mechanical binding/edge rewiring that preserves
those meanings is not a design change. Return `escalate` and require recorded
user confirmation before activating a meaning-changing candidate plan. No
mode-specific clause may narrow this predicate.

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
- Validate runtime file bindings and output-directory arguments against the
  entry point's actual path-resolution and write behavior. A binding that names
  a compatible artifact but passes it through the wrong argument or resolves
  it from the wrong working directory is **CRITICAL**. Re-audit any direct run
  adjustment that changes an argument, path, config value, or file binding.

### 3. Group Column Coverage Audit

For group comparisons:

- Open metadata from every upstream dataset.
- Inspect direct columns and keys encoded in `characteristics_ch1`.
- Verify the configured group column or a compiled per-dataset equivalent.
- When the research question names an exact contrast and upstream data contains
  additional labels, verify the compiled group-map binding declares those
  contrast labels as `allowed_values`. Evaluate group counts after applying
  that allowlist and report excluded-label counts as a warning.
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

Using the invocation's `audit_id`, generate a unique `audit_node_id` for every
compiled step and clone `source.url` into:

```text
runs/<workflow>/audit/<audit-id>/nodes/<audit-node-id>/node
```

Checkout the compiled commit in detached state, then perform this check only
against that fresh audit clone. Append lifecycle events to:

```text
runs/<workflow>/audit/<audit-id>/audit-workspace-registry.jsonl
```

Each event must include audit/node IDs, workflow and step IDs, workspace path,
registry URL, requested/observed commit, contract hash/version, release tag and
commit, command, UTC time, status, finding IDs, and disposition. Never rewrite
registry history. Never read, pull, update, rename, or reuse a shared `nodes/`
checkout; never use `git pull` during audit. A repeated audit uses a new
`audit_id` and fresh clones.

For every fresh audit clone:

- Require the compiled workflow step to record its registry URL, declared
  version, resolved default branch, exact commit SHA, and node `SKILL.md`
  SHA-256, contract version, release tag, and dereferenced release commit. A
  workflow missing any field is **CRITICAL** and must be recompiled.
- Confirm the workflow URL matches the current `registry.yaml` entry. A
  mismatch is **CRITICAL** because node provenance is ambiguous.
- Confirm the compiled version remains listed for that node in `registry.yaml`.
  It need not be the newest listed version, but an absent version is
  **CRITICAL** because the release is no longer registry-resolvable.
- Fetch tags in the fresh clone when the initial clone did not include them.
- Verify the recorded commit exists in the clone and compare detached HEAD with
  that commit. A mismatch is **CRITICAL**; append a `rejected` disposition,
  preserve the audit-node workspace and evidence, and retry with a new
  `audit_node_id` and fresh clone rather than switching a shared checkout.
- Hash the pinned node's `SKILL.md` and compare it with the compiled contract
  hash. A mismatch is **CRITICAL** because parameter provenance is ambiguous.
- Parse exactly one semantic `version` from the pinned `SKILL.md`; require it
  to equal both the compiled contract version and compiled registry version.
  Missing, malformed, duplicate, or unequal versions are **CRITICAL contract
  drift**.
- Fetch tags, resolve the recorded canonical `v<version>` or `<version>` tag,
  dereference annotated tags, and require it to equal the compiled release
  commit and pinned node commit. A missing, moved, ambiguous, or unequal tag is
  **CRITICAL release drift**. Do not repair this by checking out a different
  commit or relabeling the workflow.
- Resolve the current remote default branch through `refs/remotes/origin/HEAD`.
  If it has advanced beyond the pinned commit, report a **WARNING** for a newer
  available revision but continue auditing the pinned workflow.
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

Every candidate-plan result must identify the audit mode, candidate full-plan ID and
`based_on` plan ID, original protocol and workflow hashes, active and candidate
plan hashes, affected steps and attempts, selected and stale workspaces,
parameter differences, node replacements, dependency traversal, compatibility
evidence, verdict, and whether user confirmation is required. Only `pass` or a
fully verified `pass_with_remediation` may activate a candidate plan or
dispatch changed parameters.

Emit one NDJSON result line. The compact initial-audit examples below do not
have a workflow-run plan context. Runtime candidate-plan results include a
`plan_context` object containing every field listed above.

Pass:

```json
{"level":"result","status":"pass","checks":6,"warnings":[],"deferred":[],"critical":[]}
```

Pass with reproducible remediation:

```json
{"level":"result","status":"pass_with_remediation","audit_mode":["RERUN_PARAMETERS","NODE_REPLACEMENT","DOWNSTREAM_REPLAN"],"checks":6,"warnings":[],"deferred":[],"critical":[],"plan_context":{"candidate_plan_id":"<uuid>","based_on":"<uuid>","protocol_sha256":"<sha256>","workflow_sha256":"<sha256>","active_plan_sha256":"<sha256>","candidate_plan_sha256":"<sha256>","affected_steps":["deg","enrichment"],"affected_attempts":["<uuid>"],"selected_workspaces":[],"stale_workspaces":["<attempt-path>"],"parameter_differences":[],"node_replacements":[{"step":"deg","from":"<node-a>","to":"<node-b>"}],"dependency_traversal":["deg","enrichment"],"compatibility_evidence":["<evidence-sha256>"]},"remediated":[{"finding_id":"MF-AUD-001","type":"INPUT_ADAPTER","bundle":"<run-root>/audit-remediation/<candidate-plan-id>/MF-AUD-001/bundle/remediation.yaml","bundle_sha256":"<sha256>","verification":"pass"}],"confirmation_required":false}
```

Deferred:

```json
{"level":"result","status":"deferred","checks":6,"warnings":[],"deferred":[{"check":3,"step":"fetch1","msg":"Metadata not produced yet; run fetch steps and repeat audit."}],"critical":[]}
```

Failure:

```json
{"level":"result","status":"fail","checks":6,"warnings":[],"deferred":[],"critical":[{"check":1,"step":"fetch1","key":"rename","msg":"Unsupported required configuration."}]}
```

User decision required:

```json
{"level":"result","status":"escalate","audit_mode":["NODE_REPLACEMENT","DOWNSTREAM_REPLAN"],"plan_context":{"candidate_plan_id":"<uuid>","based_on":"<uuid>","protocol_sha256":"<sha256>","workflow_sha256":"<sha256>","active_plan_sha256":"<sha256>","candidate_plan_sha256":"<sha256>","affected_steps":["deg","enrichment"],"affected_attempts":["<uuid>"],"selected_workspaces":["<attempt-path>"],"stale_workspaces":[],"parameter_differences":[],"node_replacements":[{"step":"deg","from":"<node-a>","to":"<node-b>"}],"dependency_traversal":["deg","enrichment"],"compatibility_evidence":["<evidence-sha256>"]},"confirmation_required":true,"reason":"Replacement changes required result semantics."}
```

Execution policy:

- `fail`: execute nothing.
- `deferred`: execute only prerequisite fetch steps identified by the audit,
  then repeat the audit.
- `pass`: proceed with the remaining DAG.
- `pass_with_remediation`: embed verified remediation references in the
  complete initial or run-local candidate plan, activate that plan, and dispatch
  only through `active_plan_id`. medflow-run must verify every remediation label
  and checksum, clone the pinned base commit, apply recorded patches or run
  recorded adapters exactly, and reject any unrecorded edit.
- `escalate`: do not activate the candidate plan or dispatch the changed
  attempt until the user confirms the scientific or contract change.
