# Test Plan: model-output-budget
## Created
2026-04-26
## Goal
Verify settings-backed model output caps and context-aware request budgets for local OpenAI-compatible providers.
## Environment
- Repository: `/mnt/f/document/New project`
- Runtime: `docker run --rm -v "/mnt/f/document/New project:/repo" -w /repo/rust rust:1-bookworm`
- Deployed stack: `/mnt/f/desktop/docker/crawcode`
- Shell: bash
## Preconditions
- Branch contains the settings-backed provider/model registry.
- Docker Compose builds from the pushed feature branch.
- Test settings define `qwen3.6-35b-a3b:tr` with `maxContext: 262000` and `maxOutputTokens`.
## Steps
1. Run runtime config parsing tests for `models[].maxOutputTokens`.
2. Run runtime conversation tests for provider length stop detection.
3. Run CLI tests for bounded output-token calculation.
4. Run CLI `/context` JSON/text tests and confirm output budget fields render.
5. Run whitespace checks.
6. Push the feature branch, rebuild the Compose image, recreate the container, and verify `claw context` plus a live prompt.
## Expected Results
- Settings parser accepts positive `models[].maxOutputTokens` and rejects zero.
- Request output tokens never exceed the configured cap or available context headroom.
- `/context` JSON includes `max_output_tokens`, `next_request_output_tokens`, and `output_safety_buffer_tokens`.
- Output-limited provider turns are reported without losing partial assistant text.
- Docker-built `claw` runs with the same settings in the deployed container.
## Logs To Capture
- Cargo test summaries.
- `git diff --check`.
- Compose build/recreate result.
- `claw --version`, `claw context`, and live prompt JSON from the deployed container.
## Locator Contract
Not applicable; no browser UI.
