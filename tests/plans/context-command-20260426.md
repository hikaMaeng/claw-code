# Test Plan: context-command
## Created
2026-04-26
## Goal
Verify `/context` is a local diagnostic command that reports `models[].maxContext`, derived auto-compact threshold, buffer, and free-space categories without model calls.
## Environment
- Repository: `/mnt/f/document/New project`
- Runtime: `docker run --rm -v "/mnt/f/document/New project:/repo" -w /repo/rust rust:1-bookworm`
- Shell: bash
## Preconditions
- Branch contains the `/context` implementation.
- Test settings define `qwen3.6-35b-a3b:tr` with `maxContext: 262000`.
## Steps
1. Run `cargo test -p rusty-claude-cli context -- --nocapture --test-threads=1`.
2. Run the local CLI `context` command with `--output-format json` against a temp `CLAW_CONFIG_HOME`.
3. Confirm `/context clear` is rejected by parser tests.
4. Confirm the report derives `auto_compact_threshold: 218770` and `autocompact_buffer_tokens: 43230` from `maxContext: 262000`.
## Expected Results
- Tests pass.
- No provider/model request is required.
- JSON includes `kind: "context"`, `context_window.max_context`, `auto_compact_threshold`, `autocompact_buffer_tokens`, and category rows.
## Logs To Capture
- Cargo test summary.
- CLI JSON output or relevant fields.
## Locator Contract
Not applicable; no browser UI.
