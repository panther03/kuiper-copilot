---
name: fstarmcp
description: Use the F* MCP server for interactive, incremental typechecking of F* and Pulse code
tools: Bash, Read
---

## Overview

The F* MCP server (`fstar-mcp`) is an MCP server over F*'s `--ide` protocol. It keeps a
verifying process and a lax companion process warm for each active file, so you can edit
files on disk and request incremental checks without resending the source or reloading
dependencies.

**This is the preferred tool for interactive verification.** Use `./fstar.sh` (see the
`fstarverifier` skill) for whole-file or batch checks, and the MCP server for the
edit/check loop.

Source: [FStarLang/fstar-mcp](https://github.com/FStarLang/fstar-mcp)

## Getting the Server

Look for `fstar-mcp` in `PATH` first:

```bash
command -v fstar-mcp
```

If it is present, use it directly. Only if it is missing should you build it from a
checkout of the `fstar-mcp` repository:

```bash
cargo build --release   # produces ./target/release/fstar-mcp
```

## MCP Registration

The transport is **stdio**: the client spawns the server as a subprocess, and each client
gets its own process and session namespace. There is no port and no HTTP endpoint.

### GitHub Copilot CLI

Register the server in `.copilot/mcp-config.json` at your project root:

```json
{
  "mcpServers": {
    "fstar-mcp": {
      "type": "local",
      "command": "fstar-mcp",
      "args": [],
      "tools": ["*"]
    }
  }
}
```

### Claude Code

Register the server in `.mcp.json` at your project root:

```json
{
  "mcpServers": {
    "fstar-mcp": {
      "command": "fstar-mcp",
      "args": []
    }
  }
}
```

Restart the client after adding the configuration so the server is discovered. Add
`--verbose` to `args` to log F* protocol traffic to stderr.

## Project Configuration Discovery

You normally do **not** pass F* flags to the MCP server. For each file, the server
resolves its configuration from the first available source:

1. The nearest `*.fst.config.json`, walking up toward `workspace_root` when one is given.
   `$VAR` and `${VAR}` are expanded in all string values.
2. `make <File.fst>-in` in the file's directory, splitting `--include` pairs from the
   remaining F* options.
3. Bare defaults using `fstar.exe` from `PATH`.

Kuiper and Kuiper-based projects ship a config file at the repo root that points at the
repo's `fstar.sh` wrapper, so the server inherits exactly the flags the project needs:

```json
{
  "fstar_exe": "./fstar.sh",
  "options": ["--query_cache"],
  "include_dirs": []
}
```

Relative executable paths are resolved from the configured `cwd`. If checks fail with
missing dependencies or unknown options, verify that a config file exists at the repo
root and that it names the repo's `fstar.sh`; do not work around it by passing ad-hoc
`options` to the tools.

## Workflow

1. Edit an `.fst` or `.fsti` file with normal editing tools.
2. Call `typecheck_buffer` with `file_path`. The server reads the file from disk,
   discovers its configuration, and creates or reuses a warm session.
3. Make targeted edits **below the verified prefix** and check again. The response reports
   fragment reuse, the last verified line, duration, content hash, and staleness.
4. Use `lookup_symbol` for type and definition queries via the lax companion.
   Use `get_proof_context` for proof states, and `get_status` only when you need detailed
   fragment ranges.

Sessions are implicit: they are keyed by client, canonical path, workspace, and effective
configuration. You rarely need `create_session` or `session_id`.

## Tools

| Tool | Purpose |
|------|---------|
| `typecheck_buffer` | Read and check a file, with optional lax/position mode and deadline. Creates or reuses a session implicitly. |
| `check_project` | Check selected files, or all F* files below a workspace, in source dependency order. |
| `lookup_symbol` | Query type, docs, and definition through the lax companion. |
| `get_proof_context` | Return proof states from the latest check. |
| `get_status` | Return detailed fragment ranges from the latest check. |
| `restart_solver` | Terminate wedged Z3 descendants and restart both solvers. |
| `create_session` | Explicitly warm a session without an initial full verification. |
| `update_buffer` | Add an unsaved dependency to both F* virtual file systems. |
| `list_sessions` | List sessions owned by the current client. |
| `close_session` | Close a session owned by the current client. |

### typecheck_buffer

| Argument | Type | Notes |
|----------|------|-------|
| `file_path` | string | Preferred. File to read and check. |
| `session_id` | string | Legacy explicit session ID; use `file_path` instead. |
| `content` | string | Optional unsaved-buffer override. Disk is the default source of truth. |
| `lax` | boolean | Shortcut for `kind: "lax"`. |
| `kind` | string | `full` (default), `lax`, `cache`, `reload-deps`, `verify-to-position`, `lax-to-position`. |
| `to_line`, `to_column` | integer | For the position-based kinds. |
| `timeout` | integer | Seconds; defaults to 60. |
| `workspace_root` | string | Stop config discovery at this directory. |

Returns `status` (`ok` / `error` / `partial` / `stale`), a one-line `summary`,
`content_hash`, `stale`, `timed_out`, `finished`, `reused_fragments`, `total_fragments`,
`verified_through_line`, `duration_ms`, `process_restarted`, `dependencies_changed`,
`hint`, and `diagnostics[]`.

Checks have a 60-second default deadline and return **partial progress** on expiry rather
than failing. Diagnostics are capped at 20 per file and retain F* error numbers plus
related ranges; use `get_status` for the full fragment array.

### check_project

| Argument | Type | Notes |
|----------|------|-------|
| `workspace_root` | string | Required. Root to search and to anchor relative paths. |
| `files` | string[] | Optional subset; defaults to all `.fst`/`.fsti` below the root. |
| `timeout_per_file` | integer | Seconds per file. |

Files are checked in source dependency order. Use this for a batch sweep after a
refactor; use `typecheck_buffer` for the inner loop.

### lookup_symbol

Requires `file_path`, `line`, `column`, `symbol`. Returns `kind`
(`symbol` / `module` / `not_found`), `name`, `type_info`, `documentation`, `defined_at`.
It runs on the lax companion, so it answers while a full check is still running.

### get_proof_context

Takes `file_path` (or `session_id`) and an optional `line`. With `line`, returns the proof
state beginning at that line; without it, returns all proof states from the latest check.

### restart_solver / close_session / list_sessions

`restart_solver` takes `file_path` or `session_id` and kills wedged Z3 descendants before
restarting both the full and lax solvers — prefer it over recreating a session, which
would reload all dependencies. `close_session` requires an explicit `session_id`.
`list_sessions` lists only sessions owned by the current client.

### update_buffer

Takes `session_id`, `file_path`, and `contents`. Use it to make an unsaved dependency
visible to both F* virtual file systems when a module you are editing is depended upon by
the file being checked.

## Resource and Lifecycle Settings

| Variable | Default | Meaning |
|----------|--------:|---------|
| `FSTAR_MCP_MAX_SESSIONS` | `4` | Maximum concurrent full/lax process pairs; least-recently-used idle sessions are evicted. |
| `FSTAR_MCP_IDLE_TIMEOUT` | `1800` | Idle seconds before an unowned session is swept. |
| `FSTAR_MCP_SWEEP_PERIOD` | `300` | Seconds between cleanup sweeps. |

## Tips

- **Edit on disk, then check.** `typecheck_buffer` reads from disk by default; only pass
  `content` for a buffer you have not saved.
- **Append below the verified prefix.** Edits above the last verified line invalidate the
  prefix and force a re-check; edits below reuse fragments and are much faster.
- **Lax first.** Use `lax: true` for a fast syntax/type pass before paying for SMT.
- **Position-based checks.** `kind: "verify-to-position"` with `to_line` restricts work to
  the part of a large file you are editing.
- **Pulse files need no special setup.** The `#lang-pulse` directive is handled
  automatically, and the project config supplies any required `--ext` flags.
- **Dependency changes are handled.** When source dependencies change, the next full check
  uses `reload-deps` and reports the changed files in `dependencies_changed`.
- **Watch `process_restarted`.** If it is set, F* restarted and the fragment cache was
  lost; the next check will be slow.
- **Timeouts are not failures.** A `partial` status means the deadline expired; raise
  `timeout` or check to a position instead.
