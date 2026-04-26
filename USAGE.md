# Claw Code Usage

This guide covers the current Rust workspace under `rust/` and the `claw` CLI binary. If you are brand new, make the doctor health check your first run: start `claw`, then run `/doctor`.

## Quick-start health check

Run this before prompts, sessions, or automation:

```bash
cd rust
cargo build --workspace
./target/debug/claw
# first command inside the REPL
/doctor
```

`/doctor` is the built-in setup and preflight diagnostic. Once you have a saved session, you can rerun it with `./target/debug/claw --resume latest /doctor`.

## Prerequisites

- Rust toolchain with `cargo`
- One of:
  - `ANTHROPIC_API_KEY` for direct API access
  - `ANTHROPIC_AUTH_TOKEN` for bearer-token auth
- Optional: `ANTHROPIC_BASE_URL` when targeting a proxy or local service

## Install / build the workspace

```bash
cd rust
cargo build --workspace
```

The CLI binary is available at `rust/target/debug/claw` after a debug build. Make the doctor check above your first post-build step.

## Quick start

### First-run doctor check

```bash
cd rust
./target/debug/claw
/doctor
```

Or run doctor directly with JSON output for scripting:

```bash
cd rust
./target/debug/claw doctor --output-format json
```

**Note:** Diagnostic verbs (`doctor`, `status`, `sandbox`, `version`) support `--output-format json` for machine-readable output. Invalid suffix arguments (e.g., `--json`) are now rejected at parse time rather than falling through to prompt dispatch.

### Initialize a repository

Set up a new repository with `.claw/settings.json`, `.gitignore` entries, and a `CLAUDE.md` guidance file:

```bash
cd /path/to/your/repo
./target/debug/claw init
```

Text mode (human-readable) shows artifact creation summary with project path and next steps. Idempotent — running multiple times in the same repo marks already-created files as "skipped".

JSON mode for scripting:
```bash
./target/debug/claw init --output-format json
```

Returns structured output with `project_path`, `created[]`, `updated[]`, `skipped[]` arrays (one per artifact), and `artifacts[]` carrying each file's `name` and machine-stable `status` tag. The legacy `message` field preserves backward compatibility.

**Why structured fields matter:** Claws can detect per-artifact state (`created` vs `updated` vs `skipped`) without substring-matching human prose. Use the `created[]`, `updated[]`, and `skipped[]` arrays for conditional follow-up logic (e.g., only commit if files were actually created, not just updated).

### Interactive REPL

```bash
cd rust
./target/debug/claw
```

### One-shot prompt

```bash
cd rust
./target/debug/claw prompt "summarize this repository"
```

### Shorthand prompt mode

```bash
cd rust
./target/debug/claw "explain rust/crates/runtime/src/lib.rs"
```

### JSON output for scripting

```bash
cd rust
./target/debug/claw --output-format json prompt "status"
```

### Inspect worker state

The `claw state` command reads `.claw/worker-state.json`, which is written by the interactive REPL or a one-shot prompt when a worker executes a task. This file contains the worker ID, session reference, model, and permission mode.

Prerequisite: You must run `claw` (interactive REPL) or `claw prompt <text>` at least once in the repository to produce the worker state file.

```bash
cd rust
./target/debug/claw state
```

JSON mode:
```bash
./target/debug/claw state --output-format json
```

If you run `claw state` before any worker has executed, you will see a helpful error:
```
error: no worker state file found at .claw/worker-state.json
  Hint: worker state is written by the interactive REPL or a non-interactive prompt.
  Run:   claw               # start the REPL (writes state on first turn)
  Or:    claw prompt <text> # run one non-interactive turn
  Then rerun: claw state [--output-format json]
```

## Advanced slash commands (Interactive REPL only)

These commands are available inside the interactive REPL (`claw` with no args). They extend the assistant with workspace analysis, planning, and navigation features.

### `/ultraplan` — Deep planning with multi-step reasoning

**Purpose:** Break down a complex task into steps using extended reasoning.

```bash
# Start the REPL
claw

# Inside the REPL
/ultraplan refactor the auth module to use async/await
/ultraplan design a caching layer for database queries
/ultraplan analyze this module for performance bottlenecks
```

Output: A structured plan with numbered steps, reasoning for each step, and expected outcomes. Use this when you want the assistant to think through a problem in detail before coding.

### `/teleport` — Jump to a file or symbol

**Purpose:** Quickly navigate to a file, function, class, or struct by name.

```bash
# Jump to a symbol
/teleport UserService
/teleport authenticate_user
/teleport RequestHandler

# Jump to a file
/teleport src/auth.rs
/teleport crates/runtime/lib.rs
/teleport ./ARCHITECTURE.md
```

Output: The file content, with the requested symbol highlighted or the file fully loaded. Useful for exploring the codebase without manually navigating directories. If multiple matches exist, the assistant shows the top candidates.

### `/bughunter` — Scan for likely bugs and issues

**Purpose:** Analyze code for common pitfalls, anti-patterns, and potential bugs.

```bash
# Scan the entire workspace
/bughunter

# Scan a specific directory or file
/bughunter src/handlers
/bughunter rust/crates/runtime
/bughunter src/auth.rs
```

Output: A list of suspicious patterns with explanations (e.g., "unchecked unwrap()", "potential race condition", "missing error handling"). Each finding includes the file, line number, and suggested fix. Use this as a first pass before a full code review.

## Model and permission controls

```bash
cd rust
./target/debug/claw --model qwen3.6-35b-a3b:tr prompt "review this diff"
./target/debug/claw --permission-mode read-only prompt "summarize Cargo.toml"
./target/debug/claw --permission-mode workspace-write prompt "update README.md"
./target/debug/claw --allowedTools read,glob "inspect the runtime crate"
```

Supported permission modes:

- `read-only`
- `workspace-write`
- `danger-full-access`

## Providers and Models

Model selection is settings-backed. `providers` names endpoints, and `models` maps exact model names to those providers plus a required `maxContext` context window.

```json
{
  "model": "qwen3.6-35b-a3b:tr",
  "providers": {
    "local-lmstudio": {
      "type": "openai",
      "url": "http://192.168.0.6:12345/v1"
    },
    "dashscope": {
      "type": "dashscope",
      "url": "https://dashscope.aliyuncs.com/compatible-mode/v1",
      "apiKey": "sk-..."
    }
  },
  "models": [
    {
      "name": "qwen3.6-35b-a3b:tr",
      "provider": "local-lmstudio",
      "maxContext": 262000
    },
    {
      "name": "qwen-plus",
      "provider": "dashscope",
      "maxContext": 131072
    }
  ]
}
```

Supported provider `type` values:

- `anthropic`
- `xai`
- `openai`
- `dashscope`

`apiKey` is optional for local OpenAI-compatible providers and required for hosted providers that enforce authentication.

When `--model` is omitted, both interactive `claw` and non-interactive prompt modes (`claw prompt ...`, `claw -p ...`, shorthand prompts, and piped prompts) use the merged settings `model`. A project `.claw/settings.json` value overrides the global `$CLAW_CONFIG_HOME/settings.json` value. An explicit `--model` flag always wins over settings.

Automatic compaction uses `models[].maxContext`. The trigger is calculated at roughly 83.5% of the context window, leaving a 16.5% buffer for system tools, MCP tools, deferred tool definitions, and the next turn. For a 200k context model this triggers near 167k input tokens and keeps about 33k tokens as buffer.

## FAQ

### What about Codex?

The name "codex" appears in the Claw Code ecosystem but it does **not** refer to OpenAI Codex (the code-generation model). Here is what it means in this project:

- **`oh-my-codex` (OmX)** is the workflow and plugin layer that sits on top of `claw`. It provides planning modes, parallel multi-agent execution, notification routing, and other automation features. See [PHILOSOPHY.md](./PHILOSOPHY.md) and the [oh-my-codex repo](https://github.com/Yeachan-Heo/oh-my-codex).
- **`.codex/` directories** are not part of the active runtime lookup policy. Skills, agents, commands, settings, and sessions use only the two `.claw` tiers described below.
- **`CODEX_HOME`** and other legacy assistant config homes are ignored by the active skill/agent/config lookup path.

`claw` does **not** support OpenAI Codex sessions, the Codex CLI, or Codex session import/export. If you need to use OpenAI models (like GPT-4.1), configure the OpenAI-compatible provider as shown above in the [OpenAI-compatible endpoint](#openai-compatible-endpoint) and [OpenRouter](#openrouter) sections.

## HTTP proxy support

`claw` honours the standard `HTTP_PROXY`, `HTTPS_PROXY`, and `NO_PROXY` environment variables (both upper- and lower-case spellings are accepted) when issuing outbound requests to Anthropic, OpenAI-, and xAI-compatible endpoints. Set them before launching the CLI and the underlying `reqwest` client will be configured automatically.

### Environment variables

```bash
export HTTPS_PROXY="http://proxy.corp.example:3128"
export HTTP_PROXY="http://proxy.corp.example:3128"
export NO_PROXY="localhost,127.0.0.1,.corp.example"

cd rust
./target/debug/claw prompt "hello via the corporate proxy"
```

### Programmatic `proxy_url` config option

As an alternative to per-scheme environment variables, the `ProxyConfig` type exposes a `proxy_url` field that acts as a single catch-all proxy for both HTTP and HTTPS traffic. When `proxy_url` is set it takes precedence over the separate `http_proxy` and `https_proxy` fields.

```rust
use api::{build_http_client_with, ProxyConfig};

// From a single unified URL (config file, CLI flag, etc.)
let config = ProxyConfig::from_proxy_url("http://proxy.corp.example:3128");
let client = build_http_client_with(&config).expect("proxy client");

// Or set the field directly alongside NO_PROXY
let config = ProxyConfig {
    proxy_url: Some("http://proxy.corp.example:3128".to_string()),
    no_proxy: Some("localhost,127.0.0.1".to_string()),
    ..ProxyConfig::default()
};
let client = build_http_client_with(&config).expect("proxy client");
```

### Notes

- When both `HTTPS_PROXY` and `HTTP_PROXY` are set, the secure proxy applies to `https://` URLs and the plain proxy applies to `http://` URLs.
- `proxy_url` is a unified alternative: when set, it applies to both `http://` and `https://` destinations, overriding the per-scheme fields.
- `NO_PROXY` accepts a comma-separated list of host suffixes (for example `.corp.example`) and IP literals.
- Empty values are treated as unset, so leaving `HTTPS_PROXY=""` in your shell will not enable a proxy.
- If a proxy URL cannot be parsed, `claw` falls back to a direct (no-proxy) client so existing workflows keep working; double-check the URL if you expected the request to be tunnelled.

## Common operational commands

```bash
cd rust
./target/debug/claw status
./target/debug/claw sandbox
./target/debug/claw agents
./target/debug/claw mcp
./target/debug/claw skills
./target/debug/claw system-prompt --cwd .. --date 2026-04-04
```

## Session management

REPL turns are persisted under the global config home, namespaced by workspace fingerprint:

```text
$CLAW_CONFIG_HOME/sessions/<workspace_hash>/
```

When `CLAW_CONFIG_HOME` is unset, it defaults to `$HOME/.claw`.

```bash
cd rust
./target/debug/claw --resume latest
./target/debug/claw --resume latest /status /diff
```

Useful interactive commands include `/help`, `/status`, `/cost`, `/config`, `/session`, `/model`, `/permissions`, and `/export`.

## Config file resolution order

Runtime config is loaded in this order, with later entries overriding earlier ones:

1. `$CLAW_CONFIG_HOME/settings.json` (or `$HOME/.claw/settings.json` when `CLAW_CONFIG_HOME` is unset)
2. `<repo>/.claw/settings.json`

No `.claw.json`, legacy assistant homes, `.codex`, `.config/claw`, or `settings.local.json` files are loaded by the runtime config loader.

## Skills, agents, commands, and Docker home

The active lookup tiers are also limited to:

1. `$CLAW_CONFIG_HOME/{skills,commands,agents}`
2. `<repo>/.claw/{skills,commands,agents}`

For the Docker stack at `F:\desktop\docker\crawcode`, mount only one global home:

```yaml
volumes:
  - ./workspace:/workspace
  - ./claw-home/.claw:/home/claw/.claw
environment:
  CLAW_CONFIG_HOME: /home/claw/.claw
```

Project-specific overrides belong in `./workspace/.claw/settings.json`; global providers, models, plugins, sessions, and installed skills belong in `./claw-home/.claw/`.

## Mock parity harness

The workspace includes a deterministic Anthropic-compatible mock service and parity harness.

```bash
cd rust
./scripts/run_mock_parity_harness.sh
```

Manual mock service startup:

```bash
cd rust
cargo run -p mock-anthropic-service -- --bind 127.0.0.1:0
```

## Verification

```bash
cd rust
cargo test --workspace
```

## Workspace overview

Current Rust crates:

- `api`
- `commands`
- `compat-harness`
- `mock-anthropic-service`
- `plugins`
- `runtime`
- `rusty-claude-cli`
- `telemetry`
- `tools`
