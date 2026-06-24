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

**If `file_bindings` is present:** Use it as the authoritative wiring map. `file_bindings` was produced by compile-workflow and contains exact file resolution instructions. The checklist items marked `[fb]` below can skip inference — use the binding directly.

**If `data_type_notes` is present:** Trust the compile agent's data inspection. Do not re-inspect unless something looks wrong at runtime.

### 2. Resolve Node Packages

For each step:
1. Parse `node` field: `<name>@<version>` or just `<name>` (use latest version)
2. Check `nodes/<name>@<version>/` exists
3. If missing, fetch from git: `git clone <url>` using the node's registry entry
4. **Read `SKILL.md`** to get the node manifest (parameters, entry point, env file)
5. **Read the node's entry point** (`scripts/main.R` or `scripts/main.py`):
   - Check `file_discovery` and `file_layout` in SKILL.md first — these are authoritative
   - Verify the entry point matches the SKILL.md declarations
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

For each step in topological order, you MUST generate and complete a pre-flight checklist BEFORE dispatching the node. You MUST NOT proceed to the next checklist item until the current one is confirmed.

#### Pre-flight Checklist Template

For each step, create this exact checklist and work through it:

```
□ 1. SKILL.md read: inputs=[...], outputs=[...], parameters=[...]
□ 2. Entry point read: subcommands=[...], file_discovery: {recursive, pattern, sidecar?}
□ 3. Upstream files inspected: `find <upstream-outdir> -type f` (NOT flat ls)
□ 4. File layout confirmed: nesting? sidecar files present?
□ 5. Data type checked: opened sample file, values are (integer counts | log2 | normalized)
□ 6. Method matched: using (limma | DESeq2 | edgeR) because data is (log2 | counts)
□ 7. Config complete: required params=[...], provided=[...], overrides applied=[...]
□ 8. Env ready: conda env created at runs/<workflow>/envs/<step-id>/
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
- Run `find <upstream-outdir> -type f` — NOT `ls`, NOT `ls -la`
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
- List all provided values from workflow.json `config`
- Flag missing required params
- Flag params where the default is wrong for this data/question
- Apply overrides

**8. Env ready:**
- Create env if not exists: `conda env create -f <env-file> -p runs/<workflow>/envs/<step-id>`
- Use `--override-channels` or ensure `nodefaults` in env file channels
- Never use pre-existing global conda environments

#### Dispatch and Verify

After checklist is complete:

**Dispatch:**
```bash
conda run -p runs/<workflow>/envs/<step-id> --cwd <node-dir> \
  Rscript scripts/main.R <subcommand> --arg1 val1 --outdir runs/<workflow>/<step-id>
```

**Handle result:**
- Parse NDJSON stdout → extract result line, files, metadata
- Exit code: 0 = success, non-zero = check `exceptions` in SKILL.md

**Post-flight verification (MANDATORY):**
```
□ Outputs declared vs produced: SKILL.md says [X, Y, Z], disk has [actual files]
□ Any mismatch? If yes, check: conditional output? sidecar missing? wrong subcommand?
```
- Cross-reference SKILL.md `outputs` list with `find <outdir> -type f`
- If a declared output is missing: read the node code to find the condition that produces it
- **Do not claim "node doesn't produce X" without first checking whether your setup broke a sidecar dependency**
- Report any mismatch with the specific condition found

**Wire data to downstream:**
- Read upstream SKILL.md outputs → file paths on disk
- Read downstream SKILL.md inputs → required semantic_types
- Match by: `file_layout` declarations → semantic_type → format → filename
- Pass resolved file paths (not directories) for `param_type: file`

### 6. Report

For each step: status, files produced, runtime, warnings, any output mismatches found.
Final: workflow status, output directory, key result files.

## File Resolution Protocol

1. Use `find <dir> -type f` to list ALL files recursively — never flat `ls`
2. Match by: exact filename → semantic_type → format → file content inspection
3. Respect `file_layout` declarations in SKILL.md (nesting, sidecar)
4. **Never symlink, move, or copy files** — breaks sidecar assumptions
5. If ambiguous: report options and ask user

## Anti-Patterns

- **Do NOT** symlink files into staging directories — breaks relative path assumptions in node code
- **Do NOT** guess file layout from directory structure — read the node's actual code
- **Do NOT** use global conda environments — create fresh per-run envs under `runs/<workflow>/envs/`
- **Do NOT** use default method without checking data type compatibility
- **Do NOT** skip reading the node's entry point before dispatching
- **Do NOT** assume a node doesn't produce a file just because you didn't look for it — verify outputs against SKILL.md
