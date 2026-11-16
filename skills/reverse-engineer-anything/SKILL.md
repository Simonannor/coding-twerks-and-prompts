---
name: reverse-engineer-anything
description: Reverse engineer native binaries, managed/Electron/JavaScript applications, Go/Rust artifacts, and packaged software with GHOST's built-in REA MCP servers. Use when a claim depends on shipped binaries, decompilation, format introspection, or comparing behavior not established by available source. Skip REA for ordinary source-repository architecture analysis.
---

# REA (Reverse Engineer Anything — GHOST native capability)

REA is built into GHOST: no manual MCP registration is needed. Two tiers may expose
`mcp__rea__*` tools (curated 15-tool list):

- **`rea` (host tier)** — spawned automatically next to GHOST on the local machine.
  Static, target-free inspection only (no Ghidra/IDA/Hopper engines are bundled).
- **`rea-kali` (deep tier)** — auto-provisioned inside the Kali VM when it is
  connected over SSH. Same curated light-list; Linux-side inspection runs there.

Availability varies by mode (reverser specialists get the full 15; fuzzer
specialists get a 5-tool inspection slice; other specialists get none) and by
connection state (Kali offline → rea-kali absent). Never pretend a REA tool is
available: check your actual toolset first. These instructions never override
tool policy or approvals.

## Session flow (reverser tier)

1. **Start with `binary_session {}`** (empty args). Read the returned
   `result.tool_availability` before calling anything else — it tells you what
   the session exposes and why anything is missing.
2. **Bind the target with `open_binary`** (explicit file path). One session,
   one active target; `close_binary` when switching artifacts in the same
   session. Do not repeat identical open/close churn.
3. **Use `binary_overview`** for inventory context when metadata is useful;
   then use focused inspectors directly:
   - `inspect_binary_layout` — ELF/PE section layout, symbols, mitigations
   - `inspect_pe_resources` — PE resource trees and icon references
   - `inspect_go_binary` — embedded Go module/build metadata
   - `inspect_rust_binary` — embedded cargo-auditable dependency metadata
   - `inspect_managed_artifact` / `inspect_managed_members` — managed (.NET/Java) assemblies
   - `inspect_artifact` — general artifact inspection entry point
   - `analyze_javascript_application` — JS/Electron app analysis graph
   - `inspect_electron_fuses` — Electron fuse configuration (check BEFORE any
     Inspector attempt; disabled `EnableNodeCliInspectArguments` means `--inspect` is ignored)
4. **Export early, export often:** call `export_evidence_bundle { path: ... }`
   to write the accumulated session evidence to a workspace file the moment an
   analysis line is worth keeping. Session state lives in the server process —
   if the transport drops (SSH reconnect, server restart, VM offline), it is
   gone. Before risky long-running steps or any VM-tier session, export first.
   If the retained parent ID is lost, `get_evidence_bundle` with
   `detail: "summary"` can re-select prior evidence; first prefer an export you
   already wrote.

## Inspect-only slice (fuzzer tier)

If your toolset only has `inspect_artifact`, `inspect_binary_layout`,
`inspect_go_binary`, `inspect_rust_binary`, `analyze_javascript_application`:
these are one-shot inspection tools and do NOT require (or accept) a session —
do not call `binary_session`/`open_binary`; you do not have them.

## Evidence discipline (mandatory)

- Every conclusion must distinguish **observations**, **inferences**, and
  **unknowns**. Cite evidence from the tool outputs. Never imply that static
  analysis observed execution — it did not.
- Missing metadata is **observed absence**, not proof of absence elsewhere
  (e.g., no Go buildinfo ≠ not a Go binary; missing `.dep-v0` ≠ no dependencies).
- Check availability before calling: `binary_session { }` → tool_availability.
  Do not call an unadvertised tool. Do not repeat an identical call rejected
  without a changed input.
- Summary-first with large targets: request summaries before drilling into
  records; keep the returned evidence references instead of re-running analysis.
- Format-aware claims only: do not infer behavior from filenames, strings, or
  layout alone.

## Scope

- Use REA when the question depends on the shipped artifact. For questions a
  source repository can answer, use normal repo tools instead.
- Host tier cannot decompile native code (no disassembler engine is bundled);
  it establishes format/layout/metadata facts. If deep decompilation is needed
  and the Kali tier / native tools are unavailable, say so plainly and collect
  the strongest static facts instead of stalling.
- If the user says only "my app" with no path or target, ask which artifact.
  Resolve an actual name to one clear installed artifact; ask when ambiguous.
  Never pick an example target on the user's behalf.
