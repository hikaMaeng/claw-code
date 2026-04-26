# Test Plan: claude-feature-coverage

## Created

2026-04-26

## Goal

Verify the Rust implementation coverage for Claude Code-style skills, rules, hooks, slash commands, agents, memory files, plugins, and settings cascades.

## Environment

- Workspace: `/mnt/f/document/New project`
- Branch: `codex/two-tier-claw-config`
- Shell: Bash
- Date: 2026-04-26

## Preconditions

- Repository source is present locally.
- Rust tests can run through the `rust:1-bookworm` container image.
- No network access is required for code search.

## Steps

1. Search Rust sources for rule directory loaders and Claude-style rule paths.
2. Inspect prompt context discovery for supported memory and instruction files.
3. Inspect command discovery for skills, legacy command markdown, slash command registry, and agents.
4. Inspect runtime and plugin hook enums and validators.
5. Run command crate tests for skills and agents.
6. Record unsupported Claude Code contracts separately from permission rules.

## Expected Results

- Skills and agents resolve from project `.claw` and `$CLAW_CONFIG_HOME`.
- Legacy command markdown resolves as skills through `.claw/commands`.
- Hooks are limited to `PreToolUse`, `PostToolUse`, and `PostToolUseFailure`.
- Claude-style rule directories are absent unless a loader is found by source search.
- Existing command tests pass.

## Logs To Capture

- `rg` results for rule paths.
- Source snippets for prompt context discovery and hook events.
- `cargo test -p commands skills -- --nocapture --test-threads=1`
- `cargo test -p commands agents -- --nocapture --test-threads=1`

## Locator Contract

Not applicable. This is a Rust source and command-line verification task with no browser UI.
