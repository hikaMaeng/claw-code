# Test Plan: connected-provider-label

## Created

2026-04-26

## Goal

Verify that the REPL connected line reports the provider selected through `settings.json` model registry rather than model-name heuristics.

## Environment

- Workspace: `/mnt/f/document/New project`
- Branch: `codex/two-tier-claw-config`
- Shell: Bash
- Rust test container: `rust:1-bookworm`

## Preconditions

- `settings.json` can define `providers`, `models`, and `model`.
- OpenAI-compatible providers may omit `apiKey` for local endpoints.

## Steps

1. Create a temporary `$CLAW_CONFIG_HOME/settings.json` with model `qwen3.6-27b:mp`.
2. Link that model to provider `lmstudio`.
3. Set `providers.lmstudio.type` to `openai`.
4. Initialize `LiveCli` with the configured model.
5. Read `LiveCli::connected_line()`.
6. Run existing connected-line heuristic tests.

## Expected Results

- The configured model reports `Connected: qwen3.6-27b:mp via openai`.
- Existing Anthropic and xAI heuristic labels remain unchanged.

## Logs To Capture

- `cargo test -p rusty-claude-cli live_cli_connected_line_uses_settings_provider_type -- --nocapture --test-threads=1`
- `cargo test -p rusty-claude-cli format_connected_line -- --nocapture --test-threads=1`
- Any rustfmt availability issue.

## Locator Contract

Not applicable. This is CLI output behavior with no browser UI.
