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

First, read `nodes/manifest.yaml` for the index of available nodes, their git URLs, capabilities, and file layout conventions. This is the **only** source of node metadata.

**If `nodes/` is empty:** clone each node from its manifest URL:
```bash
git clone <url> nodes/<name>@<version>
cd nodes/<name>@<version> && git checkout <commit> && cd ../../..
```
**Never search, read, copy, or reference any directory outside this sandbox.**
`git clone` from manifest URLs is the only allowed method to obtain nodes.

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
- Map YAML keys to node parameter names (underscore → hyphen, e.g. `gse_id` → `--gse-id`)

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

### 6. Validate Data Flow

For each edge `A → B`:
- Read upstream node's outputs (from SKILL.md)
- Read downstream node's inputs (from SKILL.md)
- Confirm: does upstream produce what downstream needs?
- If gap exists (e.g., upstream produces expression matrix only, downstream also needs sample metadata):
  - Flag as data flow warning
  - Check if the upstream node can produce the missing output (conditional on a flag?)
  - Suggest config changes to enable the missing output

### 7. Generate workflow.json

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
        "subcommand": "union",
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
        "logfc_cutoff": "0.5"
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
        "prefer": "merged_expression.csv",
        "fallback": "first .csv matching 'expression'"
      },
      "--map": {
        "source_step": "merge",
        "semantic_type": "sample_metadata",
        "prefer": "merged_metadata.csv",
        "extract_group_col": "er_status",
        "note": "merged_metadata.csv has many columns; extract sample_id + er_status into two-column CSV for downstream"
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

### 8. Report

- Workflow name, step count, edge count
- Node assignments per step with versions
- Config decisions: method chosen, cutoffs applied, group column selected
- File bindings: which files flow between which steps
- Data flow warnings (if any)
- Unresolved steps (if any)
