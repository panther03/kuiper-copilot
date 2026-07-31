# Kuiper Copilot

Plugins for [Kuiper](https://github.com/FStarLang/kuiper) development — a DSL built on [F*](https://fstar-lang.org) and Pulse for programming and verifying safe GPU kernels — available for both [GitHub Copilot CLI](https://docs.github.com/copilot/concepts/agents/about-copilot-cli) and [Claude Code](https://claude.ai/code).

The canonical agent and skill files live under `plugins/kuiper-copilot/` and are shared by both ecosystems. The Copilot CLI plugin reads `plugin.json` at the repo root, which points `agents` and `skills` at `plugins/kuiper-copilot/agents/` and `plugins/kuiper-copilot/skills/`. The Claude Code plugin reads `.claude-plugin/marketplace.json` (also at the repo root), which advertises a single plugin sourced from `./plugins/kuiper-copilot/`; that directory holds `.claude-plugin/plugin.json` plus the same `agents/` and `skills/` subdirectories. The 5 skill files are loaded by both ecosystems verbatim with no duplication.

## What's Included

### Agents

| Agent | Description |
|-------|-------------|
| **fstar-coder** | An expert F* and Pulse programmer that authors formal specifications, implements solutions, and proves correctness — all proofs machine-checked. Covers pure F*, Pulse (concurrent separation logic), and the Kuiper coding and extraction rules that keep generated CUDA correct and fast. |

### Skills

| Skill | Description |
|-------|-------------|
| **smtprofiling** | Debug F* queries sent to Z3, diagnosing proof instability and performance issues. Includes a catalog of 10 proven stabilization techniques mined from real verification projects. |
| **proofdebugging** | Systematic workflows for debugging F*/Pulse verification failures — isolating failures, factoring lemmas, and hardening proofs. |
| **fstarverifier** | Verify and extract through the project Makefile, and interpret common F*/Pulse error patterns. |
| **specreview** | Review F*/Pulse specifications for completeness, strength, and usability — catch weak postconditions, missing spec-impl connections, and unusable signatures. |
| **fstarmcp** | Use the F* MCP server for interactive, incremental typechecking — the preferred tool for the edit/check loop, keeping a warm F* process per file. |

## Prerequisites

- **A Kuiper or Kuiper-based project with a Makefile.** The Makefile is the source of
  truth for verification, extraction, and compilation; the agent drives everything through
  it rather than calling `fstar.exe`, `krml`, or `nvcc` directly.
- **A project-local F*/Pulse/KaRaMeL installation.** Projects are expected to carry their
  own installation inside the repo. It is not version controlled, and its exact location
  and how to obtain it vary from project to project — refer to the project's own
  documentation. Do not build upstream F*, Pulse, or KaRaMeL by hand.
- **`./fstar.sh` and `./krml.sh` at the repo root.** These wrappers supply the
  project-specific flags for verification and extraction and delegate to the project-local
  installation, threading any additional flags through to `fstar.exe` / `krml`. `fstar.sh`
  is the escape hatch for checking a single file with an ad-hoc diagnostic flag.
- **`fstar-mcp`** (recommended) — the F* MCP server for interactive incremental
  typechecking. Look for it in `PATH` first; build it from
  [FStarLang/fstar-mcp](https://github.com/FStarLang/fstar-mcp) with `cargo build --release`
  only if it is missing. See the `fstarmcp` skill for registration and usage.

## Installation

### GitHub Copilot CLI

Requires [GitHub Copilot CLI](https://github.com/github/copilot-cli).

```bash
copilot plugin install FStarLang/kuiper-copilot
```

### Claude Code

Requires [Claude Code](https://claude.ai/code).

```
/plugin marketplace add FStarLang/kuiper-copilot
/plugin install kuiper-copilot@kuiper-copilot
```

## Usage

### GitHub Copilot CLI

#### Using the FStarCoder agent

You can invoke the agent in several ways:

1. **Via the `/agent` command** — type `/agent` in an interactive session and select `fstar-coder`.

2. **Naturally in a prompt** — mention the agent by name:
   ```
   Use the fstar-coder agent to implement a verified tiled GEMM kernel
   ```

3. **Via command line**:
   ```bash
   copilot --agent=fstar-coder --prompt "Implement a verified tiled GEMM kernel"
   ```

#### Using skills

Skills are automatically invoked when relevant, or can be called directly:

```
Use the smtprofiling skill to diagnose why this proof is slow
```

```
Use the specreview skill to check if my postconditions are strong enough
```

```
Use the proofdebugging skill to isolate this verification failure
```

### Claude Code

#### Using the FStarCoder agent

```
Use the fstar-coder agent to implement a verified tiled GEMM kernel
```

#### Using skills

Skills are invoked automatically when relevant, or explicitly:

```
Use the smtprofiling skill to diagnose why this proof is slow
```

```
Use the specreview skill to check if my postconditions are strong enough
```

```
Use the proofdebugging skill to isolate this verification failure
```

## Roadmap

Future versions will add:
- smart context retrieval with vector embeddings
- enhanced spec review with provable tests
- support for other F*-related tools, e.g. EverParse

## License

Apache 2.0 — see [LICENSE](LICENSE).
