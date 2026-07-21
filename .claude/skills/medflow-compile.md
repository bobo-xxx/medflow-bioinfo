---
name: "Compile Workflow"
description: Compile a protocol .md into workflow.json by matching steps to available node packages
category: Workflow
tags: [workflow, compile, protocol]
---

Compile an analysis protocol document into an immutable MedFlow
`workflow.json` schema `2.0`. No other workflow schema is executable.

**Input**: A protocol `.md` file path.

**Output**: `workflows/<name>.json` — a machine-readable workflow specification.

The generated workflow `name` must be a safe immutable single directory
component because audit and run use it as `<workflow>` in run-local paths.

## Steps

### 1. Parse the Protocol Document

Read the protocol `.md` file. Extract:

- **Name**: From `# Protocol: <Title>`
- **Research question**: From `## research_question` section — the biological question, grouping, expected contrasts
- **Pipeline DAG**: From the mermaid `flowchart TD` block in `## analysis_pipeline`
- **Analysis intent**: From sections such as `## analysis_preferences`,
  `## expected_agent_interpretation`, or equivalent prose
- **Step table**: From the markdown table when present. Treat it as optional;
  intent-first protocols may describe steps only through the DAG and prose.
- **Config**: From a `## config` YAML block when present. Treat it as optional;
  scientific preferences expressed in prose are not node parameters until the
  compiler maps them to a selected node contract.
- **Quality gates**: From `## quality_gates_and_veto_rules` section (if any)

### 2. Discover Available Nodes

Read `registry.yaml` only for node names, versions, and Git clone URLs. Do not
use or expect a central node manifest. Discover capabilities, subcommands,
inputs, outputs, parameters, defaults, file layouts, and exceptions by reading
each freshly cloned node's `SKILL.md` at the pinned commit.

Generate one collision-resistant `compile_id` per invocation and a unique
opaque `candidate_id` per candidate. Clone fresh from its registry URL:

```text
git clone <url> runs/compile/<compile-id>/candidates/<candidate-id>/node
```

Never search, read, copy, or reference outside the sandbox. Registry-URL
`git clone` is the only node-acquisition method. Never read, pull, update,
rename, or reuse shared `nodes/`, or put an unverified version in a directory
name. Record release identity only after the gate passes.

Append candidate ID, node/URL, clone time, observed branch/commit, contract
hash, gate result, and disposition to
`runs/compile/<compile-id>/candidate-registry.jsonl`. Preserve failures; retry
with a new candidate ID and fresh clone.

#### Release Identity Gate

Before adding a node to the candidate catalog:

1. Resolve the default branch, HEAD commit, and contract hash.
2. Require exactly one semantic `SKILL.md` frontmatter version equal to a
   registry version; never default to the first entry or `1.0.0`.
3. Fetch tags and require canonical `v<version>` or `<version>`; if both exist,
   they must dereference to the same commit.
4. Require the dereferenced tag commit to equal default-branch HEAD, not merely
   be reachable from it.
5. Record URL, registry/contract version, default branch, commit, contract hash,
   release tag, and release commit in the step source.

Any missing tag or disagreement among registry version, contract version,
release tag, and selected commit is **CRITICAL version drift**. Exclude that
node revision from compilation and report the exact observed values. Do not
guess a version from commit count, dates, branch names, or commit messages.

The resolved commit is the immutable node revision for this compiled workflow.
Every workflow step must carry this source record so audit and run can verify
and execute the same commit even if the remote default branch advances later.
Compilation never makes its inspection checkout an execution source.

For each node, read:
- `SKILL.md` frontmatter: `name`, `type`, `inputs`/`outputs` (semantic_type, format), `parameters` (bind, type, required, default), `entry_point`, `file_layout`, `file_discovery`
- **The entry point** (`scripts/main.R` or `scripts/main.py`): verify subcommands, check for conditional outputs, confirm file_discovery declarations

`SKILL.md` is the declared contract; the entry point is observed implementation
behavior. If they disagree, report contract drift and stop compilation for that
node rather than silently preferring either source.

Build an in-memory catalog from the inspected contracts:
`{name: {version, contract_version, release_tag, release_commit, url,
default_branch, commit, contract_sha256, subcommands, produces, consumes,
parameters, defaults}}`. Persist this release identity, the selected contract
hash, and the agent's node/parameter-selection reasoning in `workflow.json`.

### 3. Match Steps to Nodes

Build the semantic step list from the protocol's step table when present;
otherwise, derive it from the labeled nodes in the analysis DAG and their
descriptions in the surrounding prose.

For each semantic step:

1. If a `Tool/R Package` value is present, try an exact match against the node
   `name` field.
2. Otherwise, match the step's scientific purpose, required inputs, and desired
   results against node descriptions and semantic `consumes`/`produces`
   declarations.
3. Use pipeline position and neighboring semantic types to disambiguate
   multiple plausible nodes. Do not require the protocol author to know node
   package names.
4. If one compatible node remains, select it and record the reasoning. If the
   choice remains ambiguous, flag it for human review rather than inventing a
   package assignment.
5. Select and verify the appropriate subcommand from the chosen node's
   `SKILL.md` and entry point. The protocol does not need to name it.
6. Record step intent independently of the selected node: scientific goal,
   capability, required semantic inputs, and required semantic outputs. This
   intent is the invariant used later to audit a replacement node.

### 4. Determine Edges

Parse the mermaid DAG:
- `A[label] --> B[label]` → edge `{from: label, to: label}`
- Resolve aliases to step IDs
- Validate edges form a connected DAG (no orphan nodes, no cycles)

### 5. Extract Config Bindings

For each step, map the protocol's research question, analysis preferences, and
optional config section to node parameters only after selecting the node.

For every selected parameter, read its declaration from the pinned node
`SKILL.md` and record `value`, `source`, and `rationale` in the compiled step.
Resolve values in this order:

1. an explicit protocol value compatible with the declared type/allowed set;
2. a value deterministically implied by protocol intent or inspected upstream
   data, with the evidence recorded;
3. the node-declared default, recorded as an applied default;
4. agent-filled mechanical choice when the contract permits several equivalent
   representations, with reasoning and validation against the entry point.

Do not invent an undeclared parameter or default. If the remaining choice can
change scientific interpretation and the protocol or node contract does not
resolve it, request user direction rather than filling it silently.

Treat scientific concepts separately from literal configuration keys. For
example, "method suitable for normalized microarray measurements" is an intent
that the compiler resolves after data inspection; it is not itself a CLI
parameter name.

**From config YAML block:**
- If no config block is present, continue using research intent, analysis
  preferences, quality gates, inspected data, and node defaults.
- Read per-step key-value pairs
- Map YAML keys to node parameter names using this transform:

| Transform | Example |
|-----------|---------|
| Underscore `_` → hyphen `-` | `gse_id` → `gse-id` |
| Add `--` prefix | `gse-id` → `--gse-id` |
| Match against node's `parameters[].name` | `--gse-id` matches `--gse-id` |
| Bool params: pass as flag (no value) when `true` | `batch_correction: true` → `--batch-correction` |

Do NOT skip the transform — every YAML key goes through underscore→hyphen→prefix→match.
If the final form does not match any parameter name, flag it (caught by §6 audit gate).

**From research question:**
- If the question defines a group comparison (e.g. "ER+ vs ER-"), derive:
  - `group_col`: the metadata column to use for grouping
  - `allowed_values`: the exact group labels in the requested contrast
    (e.g. `["P", "N"]`). Add this to any group-map extraction or
    row-combination binding so unrelated, indeterminate, and missing labels
    cannot create an accidental multi-group analysis.
  - `--method`: appropriate statistical method based on:
    - Log2-transformed microarray → `limma`
    - Raw counts (integer values) → `DESeq2` or `edgeR`
    - Unknown → inspect the actual data file
- If cutoffs are specified, include them (`p_value`, `logfc_cutoff`)

**From data type inspection (when sample data available):**
- Open a sample input file and check: integer counts? log2 values? normalized?
- Set `--method` accordingly, overriding defaults if needed

**From group column resolution:**

When the research question defines a group comparison, resolve `group_col` against actual upstream metadata BEFORE assigning it to any node config. A column name present in one dataset but absent in another will silently drop samples and cause non-reproducible results.

For each upstream dataset that feeds the merge step:

1. Open the metadata CSV using the selected node `SKILL.md` file-discovery and
   sidecar declarations, confirmed against its entry point.
2. List all column names: direct columns AND `characteristics_ch1` key:value keys
3. Check whether `group_col` exists exactly

| Coverage | Action |
|----------|--------|
| Exact match in ALL upstream datasets | Assign `group_col: <name>` to the merge step config. Run agent passes `--group-col`, node produces `sample_group_map.csv` directly. |
| Mismatch — exists in some datasets, different name in others | Do NOT set `--group-col` on merge. Run the §6a equivalence search. If equivalents found in all datasets, embed `extract_group_col` with `per_dataset` in `file_bindings` for the deg step. Run agent coalesces at runtime. |
| Missing from ≥1 dataset with no equivalent found | Escalate as a data flow error. Cannot proceed — the group column cannot be resolved for all samples. |

If upstream metadata is not yet available (nodes not fetched), note `group_col` as provisional — §6a validation must be re-run after node fetch.

### 6. Validate Config Against Node Parameters

After assigning config to steps, verify each key against the target node's declared parameters (from SKILL.md):

1. For each config key on a step, check: does the node have a parameter with this name?
   - `group_col` → `--group-col` → does node's `parameters[]` contain this?
2. **If a key matches no parameter on the target node:** search other steps in the pipeline.
   - Does the key belong on a different node? (e.g., `group_col` → merge node accepts `--group-col`, deg node doesn't)
   - **Reassign** the config to the node that accepts it, and remove from the original step.
3. **If no node in the pipeline accepts the key:** flag as a config warning in `data_flow_warnings`.
   - The protocol author may have specified a config that no available node supports.
4. **If a required parameter has no config value:** check the node's `default` field. If no default, flag as an error — the run will fail.

**Config audit gate — review all warnings before writing `workflow.json`:**

After validation, review every warning and error. Categorize each:

| Severity | Pattern | Action |
|----------|---------|--------|
| **CRITICAL** | Config key mapped to no node parameter in the pipeline | **HALT.** Do not write `workflow.json`. Report the unresolved key with its step and protocol source. |
| **OK** | Config key reassigned to a different step (matched there) | Note in workflow and proceed. |
| **OK** | Required parameter missing but has a default | Note the default used and proceed. |
| **OK** | Conditional output absent because its flag was not passed | Treat the omission as expected and proceed. |

**If any CRITICAL warning exists, do not write `workflow.json`.** Halt and
report all critical warnings. The protocol config must be fixed before
compilation can succeed.

### 6a. Validate Group Column Coverage

When a research question specifies a group column (e.g., `group_col: er_status`), verify it exists across ALL upstream datasets. A column that only exists in one dataset will silently reduce sample counts and produce inconsistent results.

**For each upstream dataset (fetch step):**

1. Open the metadata CSV and list all column names
2. Check if the specified group column exists:
   - If present: count non-NA P/N values → record coverage
   - If absent: search for equivalent columns

**Finding equivalent columns (when the specified column is missing):**
- Search columns whose non-NA values match the same pattern: same set of distinct values (e.g., `{P, N, I}`)
- Search columns whose name shares a common substring with the specified column (e.g., `er_status` in `er_status_ihc`)
- Score candidates by: value match (exact = high) + name similarity + non-NA count
- The best candidate is the equivalent column for that dataset

**If equivalent column found in some datasets:**
- Flag as data flow warning: `"Column 'er_status' exists in GSE20194 (278 samples) but not GSE25066. GSE25066 has 'er_status_ihc:ch1' (502 samples, values={P,N}). Columns represent the same biology — coalescing into unified 'er_status'."`
- Embed a dataset-oriented extract directive in `file_bindings` so the run agent uses the exact column name per dataset — no guessing:
  ```json
  {
    "extract_group_col": {
      "unified_name": "er_status",
      "per_dataset": {
        "GSE25066": {"column": "er_status_ihc:ch1", "values": ["P", "N"]},
        "GSE20194": {"column": "er_status:ch1", "values": ["P", "N"]}
      }
    }
  }
  ```

**If no equivalent column found AND coverage < 50% of samples:**
- Escalate as error: `"Group column 'er_status' only covers 278/786 samples (35%). No equivalent column found in GSE25066. Cannot proceed — check protocol config."`

**If the group column exists in all datasets with full coverage:**
- No warning, proceed normally

### 7. Validate Data Flow

For each edge `A → B`:
- Read upstream node's outputs (from SKILL.md)
- Read downstream node's inputs (from SKILL.md)
- Confirm: does upstream produce what downstream needs?
- If gap exists (e.g., upstream produces expression matrix only, downstream also needs sample metadata):
  - Flag as data flow warning
  - Check if the upstream node can produce the missing output (conditional on a flag?)
  - Suggest config changes to enable the missing output

For each required node input with no valid upstream producer:

- Determine whether it is an external reference resource, such as a GMT gene-set
  database, rather than an analysis artifact that the workflow should produce.
- If it is external, add an `external_inputs` entry containing the downstream
  parameter, source URL or acquisition instruction, database/release, checksum
  when available, expected format, and run-local destination.
- Do not invent an upstream edge or silently search arbitrary local directories.
- If licensing, authentication, or an unresolved scientific choice prevents
  deterministic acquisition, stop and request the missing decision before
  declaring the workflow execution-ready.
- A missing expression matrix, sample annotation, model artifact, or other
  workflow-generated result is not an external reference resource; treat that
  as a data-flow error.

When a protocol restricts a downstream analysis to a candidate feature set but
the selected node accepts only an expression matrix, compile an explicit
`filter_features` transformation. Bind the original expression matrix plus the
upstream candidate table, declare the identifier columns, preserve candidate
order or document the chosen order, and write a run-local filtered matrix. Do
not invent an unsupported gene-list CLI parameter.

### 8. Generate workflow.json

The workflow.json must use `schema_version: "2.0"`. Reject any request to emit
or execute schema `1.0` or `1.1`; the protocol must be recompiled instead.

The workflow.json must be **fully bound**: step intent, node assignment, immutable source
revision, configuration, file bindings, and external inputs are resolved.
medflow-run must still read each cloned `SKILL.md` and entry point to verify
that the pinned implementation has not drifted from its declared contract.

For each step, produce a `config` block that covers:

- **Subcommand**: which action to invoke
- **Required config params**: all `bind: config` parameters that are required
- **Method overrides**: when the node's default doesn't match the data type
- **Cutoff overrides**: when defaults don't fit the research question or sample size
- **Group specification**: which column to use, how to map labels
- **Immutable source**: registry URL, declared version, resolved default branch,
  exact commit SHA, pinned `SKILL.md` SHA-256, contract version, matching
  release tag, and dereferenced release commit
- **Parameter provenance**: protocol/default/inspection/agent-filled source and
  rationale for every value
- **Step intent**: capability, scientific goal, and required semantic inputs
  and outputs, independent of the initially selected node

Do not compile `outdir`, an attempt ID, a selected workspace, a rerun policy, or
any other runtime state. `medflow-run` supplies the root of a fresh UUIDv4
node-attempt workspace as `--outdir`.

For each edge, produce `file_bindings` that tell the run agent exactly how to wire data:

- **Which upstream step** produces the file
- **Which file pattern** to look for (from upstream SKILL.md outputs)
- **Which downstream param** it maps to (from downstream SKILL.md inputs)
- **How to resolve** if the file needs transformation (group column extraction, etc.)

For every external reference resource, produce an `external_inputs` binding
with enough provenance and integrity information for medflow-run to acquire or
verify the exact resource without guessing.

Write to `workflows/<name>.json`:

```json
{
  "schema_version": "2.0",
  "name": "<kebab-case-name>",
  "description": "<research question summary>",
  "research_question": "Compare ER+ vs ER- breast cancer tumors to identify differentially expressed genes",
  "source_protocol": {
    "path": "protocols/<source-protocol>.md",
    "sha256": "<protocol-sha256>"
  },
  "steps": [
    {
      "id": "fetch1",
      "intent": {
        "capability": "dataset-acquisition",
        "scientific_goal": "Acquire the first discovery cohort",
        "required_inputs": [],
        "required_outputs": ["expression_matrix", "sample_metadata"]
      },
      "node": "geo-microarray-processing@1.0.0",
      "source": {
        "url": "https://github.com/bioinfo-skillful/medflow-geo-microarray.git",
        "version": "1.0.0",
        "default_branch": "main",
        "commit": "<40-hex-sha>",
        "contract_sha256": "<skill-md-sha256>",
        "contract_version": "1.0.0",
        "release_tag": "v1.0.0",
        "release_commit": "<40-hex-sha>"
      },
      "config": {
        "subcommand": "fetch",
        "gse_id": "GSE25066"
      },
      "config_provenance": {
        "subcommand": {"source": "agent-filled", "rationale": "Selected node action for protocol fetch intent."},
        "gse_id": {"source": "protocol", "rationale": "Discovery cohort named by the protocol."}
      }
    },
    {
      "id": "fetch2",
      "intent": {
        "capability": "dataset-acquisition",
        "scientific_goal": "Acquire the second discovery cohort",
        "required_inputs": [],
        "required_outputs": ["expression_matrix", "sample_metadata"]
      },
      "node": "geo-microarray-processing@1.0.0",
      "source": {
        "url": "https://github.com/bioinfo-skillful/medflow-geo-microarray.git",
        "version": "1.0.0",
        "default_branch": "main",
        "commit": "<40-hex-sha>",
        "contract_sha256": "<skill-md-sha256>",
        "contract_version": "1.0.0",
        "release_tag": "v1.0.0",
        "release_commit": "<40-hex-sha>"
      },
      "config": {
        "subcommand": "fetch",
        "gse_id": "GSE20194"
      },
      "config_provenance": {
        "subcommand": {"source": "agent-filled", "rationale": "Selected node action for protocol fetch intent."},
        "gse_id": {"source": "protocol", "rationale": "Discovery cohort named by the protocol."}
      }
    },
    {
      "id": "merge",
      "intent": {
        "capability": "cohort-harmonization",
        "scientific_goal": "Create one batch-aware discovery matrix",
        "required_inputs": ["expression_matrix", "sample_metadata"],
        "required_outputs": ["expression_matrix", "sample_group_map"]
      },
      "node": "batch-correction@1.0.0",
      "source": {
        "url": "https://github.com/bioinfo-skillful/medflow-batch-correction.git",
        "version": "1.0.0",
        "default_branch": "main",
        "commit": "<40-hex-sha>",
        "contract_sha256": "<skill-md-sha256>",
        "contract_version": "1.0.0",
        "release_tag": "v1.0.0",
        "release_commit": "<40-hex-sha>"
      },
      "config": {
        "subcommand": "intersect",
        "group_col": "er_status",
        "pattern": "expr_gene_*.csv"
      },
      "config_provenance": {
        "subcommand": {"source": "agent-filled", "rationale": "Selected contract action for cohort harmonization."},
        "group_col": {"source": "protocol", "rationale": "Protocol comparison field."},
        "pattern": {"source": "node-default", "rationale": "Pinned SKILL.md declared default."}
      }
    },
    {
      "id": "deg",
      "intent": {
        "capability": "differential-expression",
        "scientific_goal": "Identify ER-associated differential expression",
        "required_inputs": ["expression_matrix", "sample_group_map"],
        "required_outputs": ["differential_expression_table", "significant_gene_list"]
      },
      "node": "differential-analysis@1.0.0",
      "source": {
        "url": "https://github.com/bioinfo-skillful/medflow-differential-analysis.git",
        "version": "1.0.0",
        "default_branch": "main",
        "commit": "<40-hex-sha>",
        "contract_sha256": "<skill-md-sha256>",
        "contract_version": "1.0.0",
        "release_tag": "v1.0.0",
        "release_commit": "<40-hex-sha>"
      },
      "config": {
        "subcommand": "run",
        "method": "limma",
        "p_set": "padj",
        "logfc_cutoff": "0.5"
      },
      "config_provenance": {
        "subcommand": {"source": "agent-filled", "rationale": "Selected contract action for differential analysis."},
        "method": {"source": "data-inspection", "rationale": "Normalized log-scale microarray expression."},
        "p_set": {"source": "node-default", "rationale": "Pinned SKILL.md declared default."},
        "logfc_cutoff": {"source": "protocol", "rationale": "Protocol effect-size threshold."}
      }
    }
  ],
  "edges": [
    {"from": "fetch1", "to": "merge"},
    {"from": "fetch2", "to": "merge"},
    {"from": "merge", "to": "deg"}
  ],
  "file_bindings": {
    "merge": {
      "--indir": {
        "sources": ["fetch1", "fetch2"],
        "resolve": "data_root",
        "note": "Node uses recursive=TRUE with pattern=expr_gene_*.csv"
      }
    },
    "deg": {
      "--mat": {
        "source_step": "merge",
        "semantic_type": "expression_matrix",
        "prefer": "shared_expression.csv"
      },
      "--map": {
        "source_step": "merge",
        "prefer": "sample_group_map.csv",
        "allowed_values": ["P", "N"]
      }
    }
  },
  "data_type_notes": {
    "expression_data": "log2-transformed microarray (GPL96 Affymetrix)",
    "sample_count": 786,
    "groups": {"P": "ER-positive", "N": "ER-negative"}
  },
  "data_flow_warnings": [],
  "compiled_at": "<ISO timestamp>"
}
```

The compiled workflow is immutable after creation. `medflow-run` copies its
complete executable content into the first immutable run-local plan snapshot.
Runtime parameter changes, attempts, selections, replacements, and downstream
replanning belong only to `workflow-run.json` and run-local full plan snapshots.

### 9. Run medflow-audit

After writing workflow.json, load and run `medflow-audit` skill.
- If audit passes: proceed to medflow-run.
- If audit returns `pass_with_remediation`: require the generated
  `workflows/<name>.audited.json` to reference labeled remediation bundles and
  preserve the original compiled workflow SHA-256. At workflow start,
  `medflow-run` copies its complete executable content into the immutable
  initial run-local plan and dispatches only through `active_plan_id`.
- If audit fails with CRITICAL violations: halt. Fix the issues before execution.
- If audit returns DEFERRED checks: medflow-run may execute only the
  prerequisite fetch steps named by the audit. Repeat medflow-audit on the
  fetched outputs before merge, DEG, enrichment, or filtering.

### 10. Report

- Workflow name, step count, edge count
- Node assignments per step with versions
- Config decisions: method chosen, cutoffs applied, group column selected
- File bindings: which files flow between which steps
- Data flow warnings (if any)
- Unresolved steps (if any)
- Audit result (pass/fail)
