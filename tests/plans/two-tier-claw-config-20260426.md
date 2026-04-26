# Test Plan: two-tier-claw-config
## Created
2026-04-26
## Goal
Verify that runtime config, skills, sessions, and Docker wiring use only global `$CLAW_CONFIG_HOME` plus project `.claw`.
## Environment
Docker `rust:1-bookworm` with repository mounted at `/workspace`, `CARGO_TARGET_DIR=/tmp/claw-target`, working directory `/workspace/rust`.
## Preconditions
- Docker is available.
- No host Rust toolchain is required.
- Local Docker stack files are under `F:\desktop\docker\crawcode`.
## Steps
1. Run `cargo fmt --all`.
2. Run targeted runtime config tests for two-tier loading and project settings parsing.
3. Run targeted commands tests for skill root and MCP reporting behavior.
4. Run targeted tools tests for project settings mutation and `.claw` skill resolution.
5. Run targeted CLI integration tests for `/config` and `--resume latest`.
6. Inspect Docker compose/Dockerfile for one global `/home/claw/.claw` mount and no `.codex`/`.config/claw` mounts.
## Expected Results
- Config loader reports only global `settings.json` and project `.claw/settings.json`.
- Skills and agents discover only project `.claw` plus `$CLAW_CONFIG_HOME`.
- Managed sessions resolve under `$CLAW_CONFIG_HOME/sessions/<workspace_hash>/`.
- Docker mounts `./claw-home/.claw` to `/home/claw/.claw`.
## Logs To Capture
- Formatter result.
- Each targeted `cargo test` command and pass/fail summary.
- Docker compose path inspection.
## Locator Contract
Not applicable; no browser UI is changed.
