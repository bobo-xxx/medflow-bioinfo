# medflow-bioinfo

Agent-driven bioinformatics workflow framework. No compiled code — agents execute workflows guided by slash-command skills.

## Project Structure

```
medflow-bioinfo/
├── .claude/commands/          # Slash command skills
│   ├── compile-workflow.md    # Protocol .md → workflow.json
│   └── run-workflow.md        # Execute workflow.json
├── protocols/                 # Reference protocol .md files
├── workflows/                 # Generated workflow.json files
├── nodes/                     # Cloned node packages (git repos)
└── runs/                      # Workflow execution outputs
```

## Available Commands

| Command | Purpose |
|---------|---------|
| `/compile-workflow <protocol.md>` | Compile protocol to workflow.json |
| `/run-workflow <workflow.json> [--live]` | Execute workflow |

## Language

English — all artifacts, commit messages, and agent communication.

## Registry

Available node packages are discovered by scanning `nodes/` for `SKILL.md` files. Each node is a standalone git repository with the IRE node package format (SKILL.md, envs/, scripts/).


## Successful-Run Reproducibility Code Bundle

Every MedFlow/IRE node successful run must produce a `code/` directory inside its output directory. Treat `code/` as an always-produced output artifact and document it in `SKILL.md`/workflow contracts with semantic type `reproducibility_code_bundle`.

Required contents:

- `README.md` - what was run, how to rerun it, and which generated outputs it corresponds to.
- `run_command.sh` - a rerun command template using the same subcommand and effective parameters, with placeholders for machine-specific paths and no secrets.
- `parameters.json` - effective parameter values after defaults and config binding.
- `inputs_manifest.json` - input paths as provided, file sizes, and hashes when practical; never copy raw input data into `code/`.
- `environment.yaml` or copied `envs/*.yaml` - declared runtime environment used by the node.
- `session_info.txt` - package/session/runtime information when available.
- `scripts/` - a snapshot of the entrypoint and helper scripts actually used for the run.

Generate the bundle from actual effective runtime values, not hand-written examples. The bundle must not contain credentials, API tokens, hidden local config, raw input data, or unrelated repository files. Output validation and tests must fail when a successful run omits `code/` or required bundle files.

## Version

**Current:** 1.0.0
**Updated:** 2026-07-08

### Release Version Decision

Determine the next release version from the net public-contract difference
between the release candidate and the latest version tag already published to
the remote repository. Only incompatibility with that pushed release changes
the major version number. Unpublished commits, local or draft version tags, and
ordinary branch pushes do not establish a new release-version baseline. Do not
assign a version bump independently to each intermediate commit.

| Net change relative to last published version | Bump |
|---|---|
| Internal refactor, tests, documentation, or correction of an unreleased intermediate change | none |
| Backward-compatible bug fix that restores documented or intended behavior | patch |
| Backward-compatible addition to inputs, outputs, parameters, or behavior | minor |
| Incompatible change to a previously published contract | major |

Commit prefixes describe individual changes but do not mechanically determine
the release version. Select the highest bump required by the final net change.

### Version Synchronization

The node release version must remain consistent across all authoritative version
locations, including:

- `SKILL.md` `version:`
- `CLAUDE.md` `Current:`
- runtime version constants
- package metadata
- registry entries and release tags, when applicable

Determine the release version once, then update every synchronized location to
that version. Synchronizing version fields does not trigger an additional bump.

If authoritative version locations disagree, the release is incomplete. Resolve
the mismatch before committing, tagging, or pushing.

A protocol or extension may use an independent version only when it is
explicitly identified as separate from the node release version.

### Procedure

1. Identify the last published version tag.
2. Review the net runtime and public-contract difference from that tag.
3. Select the highest applicable bump from the table.
4. Keep the existing version when the net change requires no bump.
5. Update all synchronized version locations.
6. Update the `Updated:` date.
7. Verify that all authoritative version values agree.
8. Commit the release version as `chore: bump to X.Y.Z`.
9. Push the version commit with the release changes.
