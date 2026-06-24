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
- **Step table**: From the markdown table in `## analysis_pipeline` — Step, Method, Tool/R Package columns
- **Config**: From `## config` YAML block — per-step configuration (subcommands, parameters)
- **Quality gates**: From `## quality_gates_and_veto_rules` section (if any)

### 2. Discover Available Nodes

Scan `nodes/` directory for node packages. Each node has a `SKILL.md` with YAML frontmatter containing:
- `name`, `description`, `type`
- `inputs` / `outputs` (with `semantic_type` and `format`)
- `parameters` (with `bind`, `type`, `required`, `default`)
- `entry_point`

Build a registry: `{name: {version, manifest}}`.

### 3. Match Steps to Nodes

For each step in the protocol's step table:

1. Extract the `Tool/R Package` column value (the tool name)
2. Find the matching node in the registry by SKILL.md `name` field
3. If exact match not found, try fuzzy match: description keywords → node description
4. If still not found, mark as unresolved and flag for human review

### 4. Determine Edges

Parse the mermaid DAG:
- `A[label] --> B[label]` → edge `{from: label, to: label}`
- Resolve aliases to step IDs

### 5. Extract Config Bindings

For each step with a config block:
- Read key-value pairs from the YAML config section
- Include in the step's `config` field
- Map config keys to the node's declared parameters

### 6. Generate workflow.json

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
      "config": {"subcommand": "fetch", "gse_id": "GSE25066"}
    }
  ],
  "edges": [
    {"from": "<step-id>", "to": "<step-id>"}
  ],
  "created_at": "<ISO timestamp>"
}
```

### 7. Report

List: workflow name, step count, edge count, node assignments, any unresolved steps.
