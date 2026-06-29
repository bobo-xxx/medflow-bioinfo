---
name: "Compile Workflow"
description: Compile a protocol .md into workflow.json by matching steps to available node packages
category: Workflow
tags: [workflow, compile, protocol]
---

Compile an analysis protocol document into an executable workflow.json.

**Input**: A protocol `.md` file path.

**Output**: `workflows/<name>.json` — a machine-readable workflow specification.

## Steps

### 1. Parse the Protocol Document

Read the protocol `.md` file. Extract:

- **Name**: From `# Protocol: <Title>`
- **Research question**: From `## research_question` section — the biological question, grouping, expected contrasts
- **Pipeline DAG**: From the mermaid `flowchart TD` block in `## analysis_pipeline`
- **Step table**: From the markdown table — Step, Method, Tool/R Package columns
- **Config**: From `## config` YAML block — per-step configuration
- **Quality gates**: From `## quality_gates_and_veto_rules` section (if any)

### 2. Discover Available Nodes

First, read `registry.yaml` for git URLs and pinned commits — this is how nodes are fetched.
Then read `manifest.yaml` for semantic metadata — subcommands, produces/consumes, file_layout conventions.

**If `nodes/` is empty:** clone each node from its `registry.yaml` URL:
```bash
git clone <url> nodes/<name>@<version>
cd nodes/<name>@<version> && git checkout <commit> && cd ../../..
```
**Never search, read, copy, or reference any directory outside this sandbox.**
`git clone` from registry URLs is the only allowed method to obtain nodes.

For each node, read:
- `SKILL.md` frontmatter: `name`, `type`, `inputs`/`outputs` (semantic_type, format), `parameters` (bind, type, required, default), `entry_point`, `file_layout`, `file_discovery`
- **The entry point** (`scripts/main.R` or `scripts/main.py`): verify subcommands, check for conditional outputs, confirm file_discovery declarations

Build a registry: `{name: {version, manifest, subcommands, produces, consumes}}`.

### 3. Match Steps to Nodes

For each step in the protocol's step table:

1. Extract the `Tool/R Package` column value
2. Find the matching node by SKILL.md `name` field
3. If exact match not found, fuzzy match: step method/description → node description
4. If still unresolved, flag for human review
5. **Verify the configured subcommand exists** in the node's entry point

### 4. Determine Edges

Parse the mermaid DAG:
- `A[label] --> B[label]` → edge `{from: label, to: label}`
- Resolve aliases to step IDs
- Validate edges form a connected DAG (no orphan nodes, no cycles)

### 5. Extract Config Bindings

For each step, map the protocol's research question and config section to node parameters:

**From config YAML block:**
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

1. Open the metadata CSV (by convention, or from `manifest.yaml` `file_discovery.sidecar`)
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

5. **Config audit gate — review all warnings before writing workflow.json:**

   After validation, review every warning and error. Categorize each:

   | Severity | Pattern | Action |
   |----------|---------|--------|
   | **CRITICAL** | Config key mapped to NO node parameter in the pipeline | **HALT.** Do not write workflow.json. Report the unresolved key with its step and protocol source. |
   | **OK** | Config key reassigned to a different step (matched there) | Note in workflow, proceed. |
   | **OK** | Required parameter missing but has default | Note default used, proceed. |
   | **OK** | Conditional output absent (flag not passed) | Expected, proceed. |

   **If any CRITICAL warning exists, do NOT write workflow.json.** Halt and report all critical warnings. The protocol config must be fixed before compilation can succeed.

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
  "extract_group_col": {
    "unified_name": "er_status",
    "per_dataset": {
      "GSE25066": {"column": "er_status_ihc:ch1", "values": ["P", "N"]},
      "GSE20194": {"column": "er_status:ch1",    "values": ["P", "N"]}
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

### 8. Generate workflow.json

The workflow.json must be **execution-ready** — the run agent should be able to dispatch every step without reading node code.

For each step, produce a `config` block that covers:

- **Subcommand**: which action to invoke
- **Required config params**: all `bind: config` parameters that are required
- **Method overrides**: when the node's default doesn't match the data type
- **Cutoff overrides**: when defaults don't fit the research question or sample size
- **Group specification**: which column to use, how to map labels

For each edge, produce `file_bindings` that tell the run agent exactly how to wire data:

- **Which upstream step** produces the file
- **Which file pattern** to look for (from upstream SKILL.md outputs)
- **Which downstream param** it maps to (from downstream SKILL.md inputs)
- **How to resolve** if the file needs transformation (group column extraction, etc.)

Write to `workflows/<name>.json`:

```json
{
  "schema_version": "1.0",
  "name": "<kebab-case-name>",
  "description": "<research question summary>",
  "research_question": "Compare ER+ vs ER- breast cancer tumors to identify differentially expressed genes",
  "steps": [
    {
      "id": "fetch1",
      "node": "geo-microarray-processing@1.0.0",
      "config": {
        "subcommand": "fetch",
        "gse_id": "GSE25066",
        "outdir": "runs/<workflow>/fetch1"
      }
    },
    {
      "id": "merge",
      "node": "batch-correction@1.0.0",
      "config": {
        "subcommand": "intersect",
        "group_col": "er_status",
        "pattern": "expr_gene_*.csv",
        "outdir": "runs/<workflow>/merge"
      }
    },
    {
      "id": "deg",
      "node": "differential-analysis@1.0.0",
      "config": {
        "subcommand": "run",
        "method": "limma",
        "p_set": "padj",
        "logfc_cutoff": "0.5",
        "outdir": "runs/<workflow>/deg"
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
        "prefer": "sample_group_map.csv"
      }
    }
  },
  "data_type_notes": {
    "expression_data": "log2-transformed microarray (GPL96 Affymetrix)",
    "sample_count": 786,
    "groups": {"P": "ER-positive", "N": "ER-negative"}
  },
  "data_flow_warnings": [],
  "created_at": "<ISO timestamp>"
}
```

### 9. Report

- Workflow name, step count, edge count
- Node assignments per step with versions
- Config decisions: method chosen, cutoffs applied, group column selected
- File bindings: which files flow between which steps
- Data flow warnings (if any)
- Unresolved steps (if any)
