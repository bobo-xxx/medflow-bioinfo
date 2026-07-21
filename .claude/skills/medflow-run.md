---
name: "Run Workflow"
description: Execute or resume an audited MedFlow schema-2.0 workflow
category: Workflow
tags: [workflow, run, execute]
---

Execute a new workflow from `workflow.json` or resume an existing workflow from
`workflow-run.json`.

**Input**: `workflows/<name>.json` for a new run, or
`runs/<workflow-name>/<workflow-run-id>/workflow-run.json` for a resume. Default
is real conda execution. Use `--stub` for validation only.

In paths below, `<workflow>` means the immutable compiled workflow `name`
field, validated as a safe single directory component.

## Steps

### 1. Load Workflow

For a new run, read `workflow.json`. Require `schema_version: "2.0"`; reject
schema `1.0`, `1.1`, or a missing version and require recompilation. Validate
steps, edges, and each step's `id`, `intent`, `node`, immutable `source` record,
and `config`. The source record must contain
the registry URL, declared version, resolved default branch, and exact commit
SHA, plus the compiled `SKILL.md` contract hash, contract version, release tag,
and dereferenced release commit. A legacy workflow without an exact commit,
contract hash, release identity, or semantic step intent must be recompiled.

Create `<workflow-run-id>` from a UTC timestamp plus a short random suffix, for
example `20260715T143012Z-a4c91f`. Create the run root, atomically create
`.medflow-running.lock`, and write `workflow-run.json` with status `running`,
the source workflow path and SHA-256, and a UUIDv4 initial `active_plan_id`.
Copy the complete executable workflow into
`plans/<active-plan-id>.json`. The compiled workflow remains immutable.

For a resume, do not dispatch from `workflow.json`. Read
`workflow-run.json.active_plan_id`, verify its complete plan snapshot and
history, inspect the workflow lock, and dispatch only from that active plan.
Never calculate the active plan by replaying deltas.

**Prerequisite:** medflow-audit must run before execution.
- `pass`: proceed with the DAG.
- `pass_with_remediation`: verify all labeled remediation bundles, copy the
  audit-produced complete executable content into the initial or candidate
  run-local plan snapshot, activate it, and dispatch only from that plan.
- `fail`: do not execute any step.
- `deferred`: execute only prerequisite fetch steps identified by the audit,
  repeat medflow-audit on their outputs, and proceed only after it passes.

**If `remediations` is present:** Treat each entry as executable provenance,
not an informal suggestion. Before any affected step:

- require `label: MEDFLOW_AUDIT_GENERATED_REMEDIATION` in its
  `remediation.yaml`;
- verify the original workflow hash, pinned node base commit, raw-input hashes,
  patch/code hashes, environment record, and bundle checksum;
- reject missing labels, checksum drift, unrecorded working-tree changes, or a
  remediation that reports a scientific/contract change without recorded user
  approval;
- run only the recorded command from the recorded working directory and
  isolated environment;
- preserve raw inputs and write derived files only to the declared run-local
  remediation output directory;
- record the remediation finding ID and bundle checksum in step logs and final
  workflow provenance.

**If `file_bindings` is present:** Use it as the authoritative wiring map. `file_bindings` was produced by medflow-compile and contains exact file resolution instructions. The checklist items marked `[fb]` below can skip inference — use the binding directly.

**If `data_type_notes` is present:** Treat it as the compiler's expected data
type, then confirm it against a small runtime sample during the mandatory
pre-flight check. A mismatch requires re-audit; do not silently change methods.

**If `external_inputs` is present:** Treat it as the authoritative acquisition
and provenance map for reference resources that are not produced by workflow
nodes. Before dispatching the consuming step:

- acquire the declared resource into its run-local destination, or verify the
  declared local file if it was supplied by the user;
- verify its expected format and checksum when provided;
- record the source URL, database/release, retrieval time, and observed
  checksum;
- stop for user action when access requires unrecorded credentials, license
  acceptance, or an unresolved resource choice;
- never search outside the sandbox or substitute a different release silently.

### 2. Resolve Node Packages

For each step:
1. Parse `node` field: `<name>@<version>` and read the step's immutable `source`
   record. Do not resolve an unversioned node to "latest" at run time.
2. Read `registry.yaml` and confirm its URL matches `source.url` and the exact
   compiled version remains listed for the node. Do not replace it with a newer
   registry version at runtime.
3. **Always clone fresh from `source.url` in workflow-managed attempt source storage:**
   Generate the attempt UUID as specified under Execute, create the workspace,
   and clone into `<run-root>/sources/<attempt-uuid>/node/`. Never place a
   checkout inside the node's `--outdir` or share a mutable node checkout
   between attempts. Then detach at the compiled commit:

   ```text
   git clone <source.url> <run-root>/sources/<attempt-uuid>/node
   git -C <run-root>/sources/<attempt-uuid>/node checkout --detach <source.commit>
   git -C <run-root>/sources/<attempt-uuid>/node rev-parse HEAD
   ```

   The observed HEAD must exactly equal `source.commit`. Clone for every node
   dispatch, including retries; never reuse another workspace's checkout or
   replace the pin with the current remote HEAD.

   Never read, pull, update, rename, or reuse a shared `nodes/` checkout, and
   never run `git pull` for node acquisition. A failed clone or checkout keeps
   its workspace evidence; retry only in a new workspace ID with another fresh
   clone.

4. Read that workspace's freshly cloned `node/SKILL.md` for subcommands,
   parameters, defaults, inputs/outputs, file layout, discovery rules, and
   exceptions. Do not use or expect a central node manifest. Verify its SHA-256
   equals the compiled contract hash before using any filled parameter. Parse
   exactly one semantic `version` and require it to equal `source.version` and
   `source.contract_version`.

   Fetch tags and resolve `source.release_tag`, dereferencing annotated tags.
   Require it to equal both `source.release_commit` and `source.commit`, and
   require its name to be canonical for the contract version (`v<version>` or
   `<version>`). Any registry, contract, tag, or commit mismatch stops dispatch
   as release drift; never rewrite the workflow or select another version
   silently.

   For a declared `NODE_PATCH`, verify `node.patch` and its SHA-256, run
   `git apply --check`, apply the patch to this clean detached checkout, and
   confirm the resulting diff and file hashes exactly match
   `remediation.yaml`. Label the execution source as
   `pinned_commit_plus_audit_patch`; never execute an unrecorded dirty tree.
5. **CRITICAL: Sandbox integrity rules**
   - **Never reference, search, read, copy, rsync, or symlink from any directory outside this sandbox.** This includes `/work/run/projects/bio-13/test/IRE_product`, `../nodes`, `~/projects`, or any other path.
   - **Never use `cp -r`, `rsync`, or `ln -s` to obtain node packages** — always `git clone` from `registry.yaml` URLs.
   - **Never symlink files** — symlinks break sidecar file lookups (dirname resolution) and env file discovery.
   - If a node's git URL is unreachable, report the error and stop. Do not search for alternatives outside the sandbox.
6. **Read the node's entry point** (`scripts/main.R` or `scripts/main.py`):
   - Treat `SKILL.md` as the declared contract and the entry point as observed
     implementation behavior.
   - Verify the implementation matches declared subcommands, parameters,
     `file_discovery`, `file_layout`, and conditional outputs. If they disagree,
     stop and report contract drift; do not silently choose one.
   - **Do not guess file layout requirements from directory structure alone**

### 3. Build Execution Plan

From the DAG edges, determine execution order (topological sort):
- Nodes with no incoming edges run first
- Nodes run when all upstreams complete
- Parallel branches run concurrently where possible

### 4. Execute (Stub)

If `--stub` is set:
- Validate all nodes resolved
- Validate all required config params present
- Validate edges form a valid DAG (no cycles)
- Print execution plan: what runs in what order with what args
- Report: "Stub run complete. N steps validated. Ready for live execution."

### 5. Execute

Default mode — real conda execution:

#### Workflow Registry, Plans, and Node Attempts

The run root is:

```text
runs/<workflow-name>/<workflow-run-id>/
├── workflow-run.json
├── .medflow-running.lock
├── .medflow-mutation.lock
├── plans/<plan-uuid>.json
├── sources/<attempt-uuid>/node/
├── attempt-locks/<attempt-uuid>.lock
├── workspaces/<step-id>/<attempt-uuid>/
└── envs/
```

The workflow registry has one authoritative active plan pointer:

```json
{
  "schema_version": "2.0",
  "workflow_run_id": "20260715T143012Z-a4c91f",
  "status": "running",
  "source_workflow": {
    "path": "workflows/breast-cancer.json",
    "sha256": "<sha256>"
  },
  "active_plan_id": "6d5ab814-2744-4ee0-bfe8-4a26bc370a5d",
  "plan_history": [
    {
      "plan_id": "6d5ab814-2744-4ee0-bfe8-4a26bc370a5d",
      "reason": "Initial compiled plan",
      "audit_result": "pass"
    }
  ],
  "attempts": {},
  "selected_attempts": {},
  "cleanup_tombstones": []
}
```

Each `plans/<plan-uuid>.json` is a complete schema-2.0 executable plan. A
revised plan records `based_on`, the audit record, and the full effective
steps, edges, and bindings. `plan_history` explains supersession, but execution
never depends on replaying those history entries.

Before every node dispatch, generate an unordered UUIDv4 `attempt_id`; never
use an ordinal, timestamp, step name, or parent ID as attempt identity. Create
the attempt workspace and workflow-owned
`attempt-locks/<attempt-uuid>.lock`, then write the attempt record in
`workflow-run.json` with status `running`, active plan ID, workflow and step
identity, pinned node source, complete effective parameters, explicit selected
upstream workspaces, input hashes, environment identity, and start UTC.
Serialize registry and lock updates across parallel branches. Before every
registry, plan, selection, lock, or workspace mutation, atomically acquire the
shared `.medflow-mutation.lock` with operation `run` or `recovery`, commit
the mutation durably, and release it. Refuse mutation when another owner holds
the mutex; `medflow-run` never takes over a cleanup transaction.

Pass the attempt workspace root itself as `--outdir`. Keep workflow-agent
evidence in reserved workflow-managed locations that do not collide with the
node's declared `input/`, `tables/`, `figures/`, `cache/`, `code/`, or
`run.json` artifacts. The node alone owns the workspace `run.json`; the workflow
agent never initializes, overwrites, or finalizes it. Never create an `outputs/`
wrapper around the node workspace.

When execution exits, normalize the factual terminal status in
`workflow-run.json` to `success`,
`partial_success`, `no_result`, `failure`, or `interrupted`; finish logs and
review evidence; durably update the workflow registry; and remove the
workflow-owned attempt lock only after terminal state is durable. Do not edit
the node's `run.json` after process exit. The workspace is then immutable for
every status. No audit, review, retry, rerun, replacement, or cleanup record may
subsequently be written inside it.

If the executing agent dies, a resumed `medflow-run` may verify that execution
is no longer active, finalize the attempt as `interrupted`, and remove its
lock. `medflow-cleanup` cannot perform this recovery.

After terminal closure, record exactly one disposition in `workflow-run.json`:
`select`, `rerun`, `replace`, `halt`, or `escalate`. Node status and agent
disposition are separate. Never infer selection from timestamps, directory
order, UUID values, or process exit code. Downstream bindings read only the
explicitly selected terminal workspace recorded in the workflow registry.

Do not create another attempt while the current attempt is running. Multiple
commands, folds, parameter candidates, or model comparisons share one
workspace only when they are part of the node's declared exploration design;
the complete exploration exits as one attempt.

Environments are not workspaces and may be shared to avoid repeated package
installation. Derive an `env_id` from the node URL, pinned commit, environment
specification hash, platform, and accepted environment remediations; store it
at `<run-root>/envs/<env_id>/`. Record the shared `env_id` in every
workspace. Never share an environment when any of those inputs differ, and
never place mutable node outputs in the shared environment. Treat a shared
environment as immutable after its first dispatch. Any package installation,
repair, or dependency change creates a new environment specification hash and
therefore a new `env_id`; never mutate an environment already referenced by a
workspace.

For each step in topological order, generate and complete a pre-flight
checklist before dispatching the node. After every node attempt, complete the
mandatory agent review below. A successful process exit alone does not finish
a step. Do not dispatch a downstream step until the review passes.

#### Pre-flight Checklist Template

For each step, create this exact checklist and work through it:

```
□ 1. SKILL.md read: inputs=[...], outputs=[...], parameters=[...]
□ 2. Entry point read: subcommands=[...], file_discovery: {recursive, pattern, sidecar?}
□ 3. Upstream files inspected with a recursive file listing
□ 4. File layout confirmed: nesting? sidecar files present?
□ 5. Data type checked: opened sample file, values are (integer counts | log2 | normalized)
□ 6. Method matched: using (limma | DESeq2 | edgeR) because data is (log2 | counts)
□ 7. Config complete: required params=[...], provided=[...], overrides applied=[...]
□ 8. Env ready: compatible shared env verified at <run-root>/envs/<env-id>/
```

**Each `□` must become `✓` before dispatch. Do not proceed past an unchecked box.**

#### Checklist Item Details

**1. SKILL.md read:**
- Parse the YAML frontmatter
- List all `inputs`, `outputs`, `parameters` with their types and defaults
- Note `file_layout` and `file_discovery` if declared

**2. Entry point read:**
- Open `scripts/main.R` or `scripts/main.py`
- Find the subcommand dispatch section — list all valid subcommands
- Check `list.files` / `glob` / `Sys.glob` calls: recursive or flat?
- Check for sidecar file logic: does it look for companion files?
- Check for conditional output logic: `if (!is.null(meta))` — when does an output NOT get produced?

**3. Upstream files inspected:**
- List files recursively rather than using a flat directory listing:

  ```powershell
  Get-ChildItem -LiteralPath <upstream-outdir> -Recurse -File |
    Select-Object -ExpandProperty FullName
  ```

  ```bash
  find <upstream-outdir> -type f
  ```

- List every file with its full path
- If multiple upstreams, inspect each one separately

**4. File layout confirmed:**
- Compare SKILL.md `file_layout` against actual directory structure
- If `file_discovery.sidecar` declared, verify companion files exist
- If sidecar files are missing and they're `needed_for` an output, that output WILL be silently skipped
- **Never symlink files between directories** — this breaks sidecar lookups

**5. Data type checked:**
- Open a sample input file: `head -3` or `read.csv(..., nrows=5)`
- Check: are values integers (counts)? decimals around 0-20 (log2)? 0-1 (normalized)?
- Record the observed data type

**6. Method matched:**
- Log2 microarray data → `--method limma`
- Raw integer counts → `--method DESeq2` or `--method edgeR`
- Unknown → ask user, do not guess
- If the node's default method is wrong for this data type, override it in config

**7. Config complete:**
- List all required parameters (bind: config)
- List all provided values from the active plan's `config`
- Flag missing required params
- Flag params where the default is wrong for this data/question
- Apply overrides

**8. Env ready:**
- Check if a functional env already exists at `<run-root>/envs/<env-id>/`:
  - Windows: check `Scripts/Rscript.exe` for R or `python.exe` for Python.
  - POSIX: check `bin/Rscript` for R or `bin/python` for Python.
- If exists and the binary is executable: skip creation, use it directly.
- If missing or broken: create fresh.
  ```bash
  conda env create -f <env-file> -p <run-root>/envs/<env-id>
  ```
- After env creation, verify R packages are loadable. If a package is unavailable
  via conda (common with Bioconductor packages on Windows), install via Rscript:
  ```bash
  conda run -p <run-root>/envs/<env-id> Rscript -e \
    'if (!require("<pkg>")) { BiocManager::install("<pkg>", update=FALSE, ask=FALSE) }'
  ```
  Check each package declared in `<env-file>` before declaring env ready.
- Use `--override-channels` or ensure `nodefaults` in env file channels
- Never use pre-existing global conda environments

#### Dispatch and Verify

After checklist is complete:

**Dispatch:**
```bash
conda run -p <run-root>/envs/<env-id> --cwd <node-dir> \
  Rscript scripts/main.R <subcommand> --arg1 val1 --outdir <attempt-workspace>
```

**Handle result:**
- Parse NDJSON stdout → extract result line, files, metadata
- Exit code: 0 = success, non-zero = check `exceptions` in SKILL.md

**Post-flight verification (MANDATORY):**
```
□ Outputs declared vs produced: SKILL.md says [X, Y, Z], disk has [actual files]
□ Any mismatch? If yes, check: conditional output? sidecar missing? wrong subcommand?
```
- Cross-reference the `SKILL.md` outputs with a recursive listing of `<outdir>`
- If a declared output is missing: read the node code to find the condition that produces it
- **Do not claim "node doesn't produce X" without first checking whether your setup broke a sidecar dependency**
- Report any mismatch with the specific condition found

#### Agent Review and Post-Terminal Decision Gate

After **every** node attempt becomes terminal, the executing agent must review
it without modifying its workspace; do not ask the user to perform this routine
review. Write `<run-root>/reviews/<attempt-uuid>.json` with:

- `label: MEDFLOW_NODE_RUN_AGENT_REVIEW`, attempt UUID, workflow and step IDs,
  active plan ID and preceding relevant attempt UUID when applicable,
  node URL/version/commit, command, config hash, input hashes, start/end UTC,
  exit code, and environment identity;
- declared versus produced files, recursive file inventory and hashes, schema
  and dimension checks, sample/feature counts, non-finite or empty results,
  warnings/errors, and conditional-output explanations;
- scientific and statistical sanity checks required by the protocol and node
  contract, including contrast direction, class labels, thresholds, leakage
  guards, convergence/quality gates, and obvious degenerate results;
- figure inspection findings for every generated figure. Open each figure (and
  render PDFs when necessary); check readability, labels, legends, clipping,
  empty panels, implausible scales, and consistency with tabular results;
- downstream compatibility and the assessment: `acceptable`,
  `acceptable_with_warnings`, `rerun_recommended`,
  `replacement_recommended`, or `blocked`, with evidence and proposed action.

Preserve rendered PDF pages or review screenshots under
`<run-root>/review-evidence/<attempt-uuid>/` and include their hashes in
`agent-review.json`. Exclude `agent-review.json` itself from its file-hash
inventory to avoid a self-referential checksum.

On `acceptable` or `acceptable_with_warnings`, the agent may record disposition
`select` atomically in `workflow-run.json`; downstream bindings must read the
selected attempt workspace itself. Do not use a symlink, an implicit "latest"
workspace, or overwrite an earlier workspace. Use
`acceptable_with_warnings` when an output is unexpected but remains valid under
the protocol, node contract, quality gates, and downstream input contract.
Record each accepted warning, the observed evidence, expected behavior, reason
it is acceptable, and any limitation on interpretation. Conditional outputs
that are legitimately absent and non-fatal package/service warnings belong in
this category when their documented conditions are satisfied. Do not rerun
merely to eliminate a cosmetic or scientifically acceptable warning.

On `rerun_recommended`, diagnose the finding after the current workspace is
closed, record disposition `rerun`, and create a new UUIDv4 attempt workspace.
Re-run the full agent review after the rerun.
Permitted automatic corrections include environment
repair, contract-preserving configuration or file-binding correction, and the
labeled reproducible audit remediations defined by `medflow-audit`. Never edit
raw inputs or silently change biological contrast, statistical method,
normalization, identifier meaning, imputation, feature-selection meaning,
model target, output schema, or the public node contract.

Every effective argument, path, config, parameter, or file-binding correction
must use `medflow-audit` mode `RERUN_PARAMETERS` and must appear in a new
candidate full plan snapshot before dispatch. Write
`<run-root>/adjustments/<new-attempt-uuid>.json` as supporting provenance.
Record the prior
review path/hash, correction type and rationale, exact old/new values, workflow
and config hashes, command, environment change, affected inputs, and the agent
identity/time. The adjustment record never authorizes an override absent from
the active plan. A pure parameter/config/binding correction does not require a
remediation bundle; if it generates code, transforms data, changes an
environment, rewrites binding logic, or edits a node, it must additionally use
the applicable audit remediation. Include accepted adjustments in final
workflow provenance so the accepted invocation can be reproduced without
inference.

If a rerun changes effective parameters, invoke `medflow-audit` in
`RERUN_PARAMETERS` mode and dispatch only after the focused audit passes and its
candidate full plan is active. Adapter/helper code, environment changes, or a
node edit additionally require a `MEDFLOW_AUDIT_GENERATED_REMEDIATION` bundle
and every applicable audit mode. Record the complete effective parameters,
structured difference, reason, audit evidence, plan ID, and new attempt UUID in
`workflow-run.json`.

On `replacement_recommended`, record disposition `replace`, search only
`registry.yaml` for a node satisfying the immutable step intent, and invoke
`medflow-audit` with every applicable mode: `NODE_REPLACEMENT`,
`RERUN_PARAMETERS`, and/or `DOWNSTREAM_REPLAN`. A replacement always receives a
new attempt workspace and environment identity. Write one new complete UUIDv4
candidate plan snapshot with `based_on` equal to the current active plan.
Activate it only after the composed audit passes; never execute a delta-only
revision.

Plan activation and selection invalidation are one atomic registry operation.
When any revised plan becomes active, clear `selected_attempts` for each
changed step and its affected downstream subgraph, then mark those prior
attempts stale before any dispatch. A selected attempt is
usable only when its plan ID, node source, effective parameters, and bindings
match the active plan.

When a new upstream attempt is selected, mark every downstream attempt that
consumed the superseded selection stale. Preserve those workspaces unchanged.
For each affected step, the agent must rebind, rerun, replace, halt, or
escalate, and `medflow-audit` must verify the complete revised downstream
subgraph before activation. Independent branches may continue only when they
have no data, quality-gate, or scientific dependency on the blocked path. A
global gate or required core-step failure pauses the whole workflow.

Across all audit and correction paths, a meaning-changing change is one that
alters specifications, approved scientific design or declared workflow intent,
scientific intent or objectives, public contracts, required result semantics,
protected assumptions, or downstream interpretation. A contract-compatible
implementation substitution or mechanical binding/edge rewiring that preserves
those meanings is not a design change. A meaning-changing change requires user
confirmation before candidate-plan activation.

Continue the diagnose-adjust-rerun or replacement loop while a new safe
corrective action is available. Record `halt` or `escalate` and stop the
affected downstream path when:

- the correction would change scientific meaning or the public contract and
  needs user confirmation;
- required data, credentials, licensed resources, or network access remain
  unavailable after applicable recovery attempts;
- the node cannot be made reproducible in the sandbox; or
- the same failure persists for three consecutive attempts with no new
  evidence or safe corrective action.

Never treat a warning as an automatic failure, or a zero exit code or existing
output files as an automatic selection. Report all attempts and dispositions.

**Inline transformation (when needed):**

If `file_bindings` for this step contains `extract_group_col` or `transform` directives:
1. Read the source file identified in the binding
2. Execute the transformation inline — write a minimal script, not a new node:
   - When the binding declares `allowed_values`, retain only rows whose group
     value is in that exact allowlist. Report counts for retained and excluded
     values. Apply this rule to direct extraction, coalescing, and
     `combine_rows` transformations.
   - **Single column:** `extract_group_col: "er_status"` → extract `sample_id` + group column:
     ```python
     import csv
     with open("merged_metadata.csv") as inf, open("sample_group_map.csv", "w") as outf:
         r = csv.DictReader(inf); w = csv.writer(outf)
         w.writerow(["sample_id", "group"])
         for row in r: w.writerow([row["geo_accession"], row["er_status"]])
     ```
   - **Coalesce with fallback chain (deterministic — compiled by medflow-compile):**
     ```json
     {
       "extract_group_col": {
         "unified_name": "er_status",
         "sources": {"er_status": ["P", "N"], "er_status_ihc:ch1": ["P", "N"]},
         "fallback_chain": ["er_status", "er_status_ihc:ch1"]
       }
     }
     ```
     → Try each column in `fallback_chain` order. For each row, use the first column that has a valid value (in `sources[column]` allowlist). Filter to rows with valid groups. Report unified counts.
   - `transform: "transpose"` → transpose matrix
   - `transform: "merge_columns"` → combine multiple columns
   - `transform: "filter_features"` → subset an expression matrix by an
     upstream candidate table using the binding's declared feature-ID columns.
     Report candidate count, matched count, missing identifiers, duplicates,
     and final matrix dimensions; never substitute fuzzy identifier matches.
3. Write the output file to the current step's outdir
4. Bind the transformed file to the downstream parameter
5. Report: "Inline transform: coalesced er_status + er_status_ihc:ch1 → sample_group_map.csv (P=461, N=319)"

When the transformation is supplied by an audit `INPUT_ADAPTER` or
`HELPER_CODE` bundle, use the recorded script and command rather than rewriting
it. Verify source-input hashes before execution and derived-output hashes after
execution, and include the remediation label/finding ID in the step report.

**Wire data to downstream:**
- Read `file_bindings` only from the active full plan snapshot. Use the declared
  source step, selected attempt workspace, preferred artifact, and transform.
- If no file_bindings: read upstream SKILL.md outputs vs downstream inputs, match by semantic_type
- Pass resolved file paths (not directories) for `param_type: file`

### 6. Report

For each step, report every attempt UUID, factual terminal status, disposition,
effective parameter differences, node replacement, active plan revision,
selected workspace, stale dependents, environment identity, outputs, warnings,
and review evidence.

When no attempt is running, atomically write the workflow's terminal or paused
status and remove the workflow lock only after the registry is durable. Report
the run root, active plan ID, selected workspaces, blocked or stale paths, key
results, and all user-confirmation decisions. Never call `medflow-cleanup`
automatically, including after success, failure, interruption, or replacement.

## File Resolution Protocol

1. List all files recursively with `Get-ChildItem -Recurse -File` on
   PowerShell or `find <dir> -type f` on POSIX; never use a flat listing.
2. Match by: exact filename → semantic_type → format → file content inspection
3. Respect `file_layout` declarations in SKILL.md (nesting, sidecar)
4. **Never symlink, move, or copy files** — breaks sidecar assumptions
5. If ambiguous: report options and ask user

## Anti-Patterns

- **Do NOT** symlink files into staging directories — breaks relative path assumptions in node code
- **Do NOT** guess file layout from directory structure — read the node's actual code
- **Do NOT** use global conda environments; use identity-matched environments
  under `<run-root>/envs/`.
- **Do NOT** dispatch schema 1.x workflows or resume from the original
  `workflow.json`; load `workflow-run.json.active_plan_id`.
- **Do NOT** reuse or modify a terminal attempt workspace.
- **Do NOT** infer order or selection from UUIDs, timestamps, or directory names.
- **Do NOT** invoke cleanup automatically or clear a stale lock.
- **Do NOT** use default method without checking data type compatibility
- **Do NOT** skip reading the node's entry point before dispatching
- **Do NOT** assume a node doesn't produce a file just because you didn't look for it — verify outputs against SKILL.md
