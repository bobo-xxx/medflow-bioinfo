---
name: "Run Workflow"
description: Execute a workflow.json step by step, dispatching nodes via conda
category: Workflow
tags: [workflow, run, execute]
---

Execute a workflow from a workflow.json file.

**Input**: `workflows/<name>.json` with optional `--live` flag for real conda execution (default is dry-run: validate only, no node dispatch).

## Steps

### 1. Load Workflow

Read `workflow.json`. Validate structure: steps, edges, each step has `id`, `node`, `config`.

### 2. Resolve Node Packages

For each step:
1. Parse `node` field: `<name>@<version>` or just `<name>` (use latest version)
2. Check `nodes/<name>@<version>/` exists
3. If missing, fetch from git: `git clone <url>` using the node's registry entry
4. Read `SKILL.md` to get the node manifest (parameters, entry point, env file)

### 3. Build Execution Plan

From the DAG edges, determine execution order (topological sort):
- Nodes with no incoming edges run first
- Nodes run when all upstreams complete
- Parallel branches run concurrently where possible

### 4. Execute (Dry Run)

If `--live` not set:
- Validate all nodes resolved
- Validate all required config params present
- Validate edges form a valid DAG (no cycles)
- Print execution plan: what runs in what order with what args
- Report: "Dry run complete. N steps validated. Ready for --live."

### 5. Execute (Live)

If `--live` is set:

For each step in topological order:

**a. Prepare environment:**
- Read node's `envs/env-*.yaml` (or `env.yaml` legacy)
- Create conda env at `runs/<workflow>/envs/<step-id>/` if not exists:
  ```bash
  conda env create -f <env-file> -p runs/<workflow>/envs/<step-id>
  ```
- Use the created env for execution

**b. Build CLI args:**
- From step `config` field → match to node parameters with `bind: config`
- From upstream step outputs → match to node parameters with `bind: upstream`
- Framework params (`--outdir`) → set to `runs/<workflow>/<step-id>/`

**c. Dispatch node:**
```bash
conda run -p runs/<workflow>/envs/<step-id> --cwd <node-dir> \
  Rscript scripts/main.R <subcommand> --arg1 val1 --outdir runs/<workflow>/<step-id>
```

**d. Handle result:**
- Parse NDJSON stdout → extract result line, files, metrics
- Check exit code: 0 = success, non-zero = check exceptions
- Handle gate decisions: pass → continue, caution → continue with warning, rerun → retry upstream, veto → pause

**e. Wire data to downstream:**
- Read upstream SKILL.md outputs
- Read downstream SKILL.md inputs
- Match by semantic_type → format → filename
- When directory needs file resolution: find specific files within upstream outdir
- Pass resolved file paths to downstream step

### 6. Report

Per step: status (complete/failed/skipped), files produced, runtime, any warnings.
Final: workflow status, output directory, key result files.

## File Resolution Protocol

When a downstream node expects specific files but upstream produces a directory:

1. Read upstream output directory → list all files
2. Read downstream input declarations (from SKILL.md)
3. Match: semantic_type → format → filename pattern → column inspection
4. If ambiguous: report options and ask user, or escalate if agentic path unavailable
5. Bind resolved file paths to downstream args
