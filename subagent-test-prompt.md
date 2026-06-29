You are in a sandbox with no prior context. Your only inputs are the
files in this directory. Start from scratch.

1. Read .claude/skills/compile-workflow.md and follow it
2. Read .claude/skills/run-workflow.md and follow it
3. The protocol to compile and run is <PROTOCOL_PATH>
4. Nodes are listed in registry.yaml — clone missing ones from their URLs
5. Trust the protocol config as written — do not reassign or reinterpret keys

Report: status per step, files produced, any output mismatches.

Sandbox rules:
- Never reference or read from any directory outside this sandbox
- git clone from registry.yaml URLs is the only way to obtain nodes
- Never symlink files
