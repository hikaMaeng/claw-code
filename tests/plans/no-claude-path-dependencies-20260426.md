# Test Plan: no-claude-path-dependencies

## Created

2026-04-26

## Goal

Verify that Rust runtime code no longer depends on `.claude` paths, `CLAUDE_CONFIG_HOME`, `CLAUDE_CONFIG_DIR`, or `.claude-plugin` manifest directories.

## Environment

- Workspace: `/mnt/f/document/New project`
- Branch: `codex/two-tier-claw-config`
- Shell: Bash
- Rust test container: `rust:1-bookworm`

## Preconditions

- Project and global config are expected to use only `<project>/.claw` and `$CLAW_CONFIG_HOME`.
- Anthropic API provider support remains valid and is out of scope for `.claude` filesystem path removal.

## Steps

1. Search Rust crates for `.claude`, `CLAUDE_CONFIG_HOME`, `CLAUDE_CONFIG_DIR`, `claude-prompt-cache`, and `.claude-plugin`.
2. Move prompt cache roots to `$CLAW_CONFIG_HOME/cache/prompt-cache`, `$HOME/.claw/cache/prompt-cache`, and temp `claw-prompt-cache`.
3. Move plugin manifests from `.claude-plugin/plugin.json` to `.claw-plugin/plugin.json`.
4. Remove legacy `.claude` skill and command test fixtures.
5. Run focused tests for prompt cache, command discovery, plugin manifest loading, and tools skill resolution.
6. Run `git diff --check`.

## Expected Results

- Rust source search returns no `.claude` path dependencies.
- Prompt cache tests pass using `CLAW_CONFIG_HOME`.
- Command skills/agents tests pass using `.claw` roots.
- Plugin tests pass with `.claw-plugin/plugin.json`.
- Existing `.claw` skill tool behavior remains intact.

## Logs To Capture

- `rg -n "\\.claude|CLAUDE_CONFIG_HOME|CLAUDE_CONFIG_DIR|claude-prompt-cache|\\.claude-plugin" rust/crates -g '*.rs'`
- `cargo test -p api prompt_cache -- --nocapture --test-threads=1`
- `cargo test -p commands skills -- --nocapture --test-threads=1`
- `cargo test -p commands agents -- --nocapture --test-threads=1`
- `cargo test -p plugins manifest -- --nocapture --test-threads=1`
- `cargo test -p tools skill_ -- --nocapture --test-threads=1`
- `git diff --check`

## Locator Contract

Not applicable. This is Rust source and CLI behavior with no browser UI.
