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
4. **Read `SKILL.md`** to get the node manifest (parameters, entry point, env file)
5. **Read the node's entry point** (`scripts/main.R` or `scripts/main.py`):
   - Check how it discovers input files (recursive? flat? pattern?)
   - Check if outputs are conditional on flags
   - Check what subcommands produce what outputs
   - **Do not guess file layout requirements from directory structure alone**

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

**a. Before dispatching — mandatory checks:**

1. **Read the node's SKILL.md** — check inputs, outputs, parameters, defaults, exceptions
2. **Read the node's entry point** (`scripts/main.R` or `scripts/main.py`):
   - How does it find input files? (`list.files(recursive=TRUE)`? `Sys.glob`?)
   - What subcommands exist? What does each produce?
   - Are any outputs conditional on flags?
3. **Check data type compatibility:**
   - If expression data is log2-transformed microarray → use limma (not DESeq2/edgeR)
   - If data is raw counts → DESeq2/edgeR is appropriate
   - Verify by inspecting a few rows of the actual input file
4. **Check SKILL.md defaults against the actual data and research question**
   - Default method correct for this data type?
   - Default cutoffs appropriate for the sample size?
   - Override defaults via `config` in workflow.json if needed

**b. Prepare environment:**
- Read node's `envs/env-*.yaml` (or `env.yaml` legacy)
- Prefer `envs/` directory (new format) over root `env.yaml` (legacy)
- Create conda env at `runs/<workflow>/envs/<step-id>/` if not exists:
  ```bash
  conda env create -f <env-file> -p runs/<workflow>/envs/<step-id>
  ```
- If the env file has `nodefaults` in channels and conda still tries defaults, check system conda config
- Use proxy `http://localhost:2999` if network is slow
- **Do not use pre-existing global conda environments** — create fresh per-run

**c. Build CLI args:**
- From step `config` field → match to node parameters with `bind: config`
- From upstream step outputs → match to node parameters with `bind: upstream`:
  - **Single upstream**: pass the upstream outdir directly (e.g., `--mat runs/<workflow>/merge/`)
  - **Multiple upstreams**: pass the run data root (e.g., `--indir runs/<workflow>/`)
  - **Do NOT symlink, move, or restructure upstream output files** — the node handles its own file discovery. If a node expects a specific layout, it's documented in its code.
- Framework params (`--outdir`) → set to `runs/<workflow>/<step-id>/`
- Skip null/empty config values — they represent absent optional parameters

**d. Dispatch node:**
```bash
conda run -p runs/<workflow>/envs/<step-id> --cwd <node-dir> \
  Rscript scripts/main.R <subcommand> --arg1 val1 --outdir runs/<workflow>/<step-id>
```

**e. Handle result:**
- Parse NDJSON stdout → extract result line, files, metrics
- Check exit code: 0 = success, non-zero = check `exceptions` in SKILL.md
- Handle gate decisions: pass → continue, caution → continue with warning, rerun → retry upstream, veto → pause
- If node fails with an exception declared in SKILL.md (`skip_with_warning`), record warning and continue if allowed
- If node fails undeclared → escalate to user

**f. Verify outputs exist:**
- After dispatch, check that declared output files actually exist on disk
- Cross-reference SKILL.md `outputs` list with actual files in the outdir
- Report any mismatch: "SKILL.md declares X but only Y produced"
- This catches bugs where SKILL.md and code have drifted

**g. Wire data to downstream:**
- Read upstream SKILL.md outputs
- Read downstream SKILL.md inputs
- Match by semantic_type → format → filename
- When directory needs file resolution: find specific files within upstream outdir
- Pass resolved file paths to downstream step

### 6. Report

For each step: status (complete/failed/skipped), files produced, runtime, any warnings.
Final: workflow status, output directory, key result files.

## File Resolution Protocol

When a downstream node expects specific files but upstream produces a directory:

1. Read upstream output directory → list all files recursively
2. Read downstream input declarations (from SKILL.md)
3. Match by: exact filename → semantic_type → format → file content inspection
4. Prefer specific files over directories when `param_type: file`
5. **Do not move, symlink, or copy files** to force matches — pass the discovered paths directly
6. If ambiguous: report options and ask user, or escalate if agentic path unavailable
7. Bind resolved file paths to downstream args

## Anti-Patterns

- **Do NOT** symlink files into staging directories — breaks relative path assumptions in node code
- **Do NOT** guess file layout from directory structure — read the node's actual code
- **Do NOT** use global conda environments — create fresh per-run envs under `runs/<workflow>/envs/`
- **Do NOT** use default method without checking data type compatibility
- **Do NOT** skip reading the node's entry point before dispatching
- **Do NOT** assume a node doesn't produce a file just because you didn't look for it — verify outputs against SKILL.md
