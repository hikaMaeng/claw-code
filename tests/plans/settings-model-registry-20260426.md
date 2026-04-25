# Test Plan: settings-model-registry
## Created
2026-04-26

## Goal
Verify settings-backed provider/model registry parsing, model-context-driven auto compaction, and CLI/API dispatch compatibility.

## Environment
Docker `rust:1-bookworm` with repository mounted at `/workspace`, `CARGO_TARGET_DIR=/tmp/claw-target`, working directory `/workspace/rust`.

## Preconditions
Repository contains Rust workspace dependencies in `rust/Cargo.toml`. No provider network credentials are required for unit and integration tests.

## Steps
1. Run `cargo test -p runtime -- --nocapture`.
2. Run `cargo test -p api -- --nocapture`.
3. Run `cargo test -p rusty-claude-cli -- --nocapture`.
4. Run targeted regression `cargo test -p api settings_provider_builds_openai_client_without_env_routing -- --nocapture`.

## Expected Results
Runtime config parses `providers` and `models`. Auto compaction threshold derives from `maxContext`. API settings provider builds without `OPENAI_*` env routing for local OpenAI-compatible providers. CLI accepts bare model names and ignores `ANTHROPIC_MODEL`.

## Logs To Capture
Command exit code, package test totals, failed test names if any, ignored live-network tests.

## Locator Contract
Not applicable; no browser UI.
