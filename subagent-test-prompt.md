You are in a clean sandbox with no prior context. Start from scratch.

Compile, audit, and run the MedFlow protocol at
`protocols/9-node-breast-cancer.md` using the protocol configuration and
scientific intent as written.

## Contract-driven compilation

- Use `registry.yaml` only to discover node names, declared versions, and Git
  clone URLs. There is no central node manifest.
- Clone every candidate or required node fresh from its registry URL. Never
  reuse an existing node checkout or historical compiled workflow.
- For each clone, resolve the remote default branch and exact commit SHA. Read
  the pinned node's `SKILL.md` as its public contract and verify the contract
  against the declared entry point implementation.
- Select node packages, subcommands, parameters, defaults, bindings, expected
  outputs, conditional outputs, and external inputs from the protocol plus the
  inspected node contracts. Do not expect the protocol to contain node-specific
  command syntax or filenames.
- Record the node URL, version, default branch, commit SHA, `SKILL.md` SHA-256,
  node-selection rationale, and every filled parameter's value, source, and
  rationale in the compiled workflow.
- Do not invent parameters or defaults. Ask only when an unresolved choice
  changes scientific interpretation; fill mechanical contract-compatible
  choices autonomously and record the reasoning.

## Audit and remediation

Run `medflow-audit` before unrestricted execution. If the audit is deferred,
run only the prerequisite fetch steps named by the audit, then repeat the
affected checks.

The audit may make contract-preserving environment fixes, deterministic input
adapters, helper code, or node patches. Preserve every raw input unchanged.
For every repair, create and verify a complete
`MEDFLOW_AUDIT_GENERATED_REMEDIATION` bundle containing its finding ID, pinned
base revision, original workflow checksum, source code or complete patch,
commands, environment, seeds, input/code/output hashes, tests, logs, smoke-test
result, and re-audit evidence. Run only the resulting audited workflow.

Do not autonomously change the biological contrast, statistical method,
normalization, identifier meaning, imputation policy, feature-selection
meaning, model target, output contract, or public node contract.

## Node execution workspaces

Give every individual node dispatch, including retries, a new globally unique
`workspace_id`. Create its isolated workspace beneath:

```text
runs/<workflow>/workspaces/<workspace_id>/
```

Keep the fresh pinned node checkout, exact command, inputs/configuration hashes,
logs, outputs, review evidence, and adjustments inside that workspace. Link a
retry to its parent workspace ID and append all lifecycle events to
`workspace-registry.jsonl`.

Environments may be shared only when the node URL, pinned commit, environment
specification hash, platform, and remediation state are identical. Identify a
shared environment with `env_id` and treat it as immutable after first use.

## Mandatory agent review and retry

After every node dispatch, inspect the logs, exit status, recursive file
inventory, hashes, schemas, dimensions, sample/feature counts, quality gates,
scientific sanity, downstream compatibility, and every generated figure. Write
a labeled `MEDFLOW_NODE_RUN_AGENT_REVIEW` record in that workspace with one
decision:

- `pass`: accept the workspace;
- `pass_with_warnings`: accept unexpected but contract-valid results, recording
  evidence, rationale, and interpretation limits;
- `retry`: diagnose, make the smallest reproducible contract-preserving
  correction, and rerun in a new workspace;
- `blocked`: stop when no new safe corrective action remains or user authority
  is required.

Do not treat warnings as automatic failures or exit code zero as an automatic
pass. Continue the diagnose-adjust-rerun loop while a new safe correction is
available. Route generated code, data transformations, binding-logic rewrites,
or node edits through audit remediation and re-audit.

Persist the accepted choice atomically in
`workspace-selections/<step-id>.json`. Downstream steps may consume only that
selected workspace and recorded selection revision. Never infer the accepted
workspace from timestamps, directory ordering, or the largest attempt number.

## External resources

Acquire only resources declared by compiled `external_inputs`. Record their
source, database/release, retrieval time, destination, and observed checksum.
Stop rather than guess when credentials, licensing, release identity, or a
scientific resource choice is unresolved.

## Final report

Report:

- interpreted scientific intent, workflow DAG, assumptions, and filled choices;
- every node URL, version, default branch, commit and contract hash;
- audit status, deferred checks, warnings, critical findings, and remediations;
- every workspace ID, step, attempt ordinal, parent workspace ID, environment
  ID, command, review decision, adjustments, runtime, and warnings;
- selected workspace and selection revision for every completed logical step;
- every file produced, with its producing workspace and checksum;
- figure-review findings, excluded samples, failed or accepted quality gates,
  and accepted-warning rationales;
- all declared-versus-produced output mismatches and conditional absences;
- external-resource provenance;
- remediation bundle paths, labels, checksums, code/patch files, tests, logs,
  and exact reproduction commands;
- final workflow status and each unresolved failure classified as computational,
  scientific, environmental, access-related, or external-source drift.

## Sandbox rules

- Never reference, search, read, copy, or derive data or code from a directory
  outside this sandbox.
- Git clone from a `registry.yaml` URL is the only permitted way to obtain a
  node package.
- Never reuse, copy, or symlink a node checkout from another location.
- Never create symlinks.
- Never silently substitute a node revision, dataset, external database release,
  method, contrast, group orientation, or accepted workspace.
