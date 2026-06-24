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

Scan `nodes/` directory for node packages. For each node, read both:
- `SKILL.md` frontmatter: `name`, `type`, `inputs`/`outputs` (semantic_type, format), `parameters` (bind, type, required, default), `entry_point`
- **The entry point** (`scripts/main.R` or `scripts/main.py`): which subcommands exist, what each produces, whether outputs are conditional

Build a registry: `{name: {version, manifest, subcommands}}`.

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

Write to `workflows/<name>.json`:

```json
{
  "schema_version": "1.0",
  "name": "<kebab-case-name>",
  "description": "<research question summary>",
  "steps": [
    {
      "id": "<step-id>",
      "node": "<node-name>@<version>",
      "config": {
        "subcommand": "fetch",
        "gse_id": "GSE25066"
      }
    }
  ],
  "edges": [
    {"from": "<step-id>", "to": "<step-id>"}
  ],
  "data_flow_warnings": [
    {"edge": "merge→deg", "gap": "sample_group_map not produced by upstream, config override needed"}
  ],
  "created_at": "<ISO timestamp>"
}
```

### 8. Report

- Workflow name, step count, edge count
- Node assignments per step
- Config bindings derived from research question
- Data flow warnings (if any)
- Unresolved steps (if any)
