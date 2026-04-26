# Test Plan: provider-empty-output-retry
## Created
2026-04-26
## Goal
Verify that an empty assistant stream from a provider is retried once and then classified explicitly if it stays empty.
## Environment
- Repository: `/mnt/f/document/New project`
- Runtime: `docker run --rm -v "/mnt/f/document/New project:/repo" -w /repo/rust rust:1-bookworm`
- Deployed stack: `/mnt/f/desktop/docker/crawcode`
- Shell: bash
## Preconditions
- Branch contains the two-tier settings work.
- Docker Compose builds from the pushed feature branch.
## Steps
1. Run runtime tests filtered to empty-output retry.
2. Run CLI tests filtered to error-kind classification.
3. Run `cargo fmt --check` inside Docker.
4. Run `git diff --check`.
5. Push the feature branch, rebuild the Compose image with `--no-cache --pull`, and recreate the `claw-code` container.
6. Confirm the deployed binary reports the new Git SHA and still answers local `claw context`.
## Expected Results
- The first empty stream is retried once.
- Persistent empty streams fail with `provider_empty_output`.
- The session contains no persisted empty assistant message.
- The Docker-built CLI remains healthy after deployment.
## Logs To Capture
- Cargo test summaries.
- Formatting and whitespace checks.
- Compose build/recreate result.
- `claw --version` and `claw context` output from the deployed container.
## Locator Contract
Not applicable; no browser UI.
