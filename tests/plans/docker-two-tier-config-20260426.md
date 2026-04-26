# Test Plan: docker-two-tier-config
## Created
2026-04-26
## Goal
Verify the `F:\desktop\docker\crawcode` Docker stack builds Claw from the pushed GitHub branch and preserves global/project `.claw` config precedence for model execution, WebSearch, and collisions.
## Environment
- Compose project: `/mnt/f/desktop/docker/crawcode`
- Source ref: `https://github.com/hikaMaeng/claw-code.git`, branch `codex/two-tier-claw-config`
- Image: `claw-code:local`
- Service: `claw`
## Preconditions
- Docker Compose is available.
- External Docker network `linker` exists.
- Global settings file exists at `claw-home/.claw/settings.json`.
- Project settings file exists at `workspace/.claw/settings.json`.
## Steps
1. Validate compose syntax with `docker compose config`.
2. Build the image with `docker compose build --no-cache claw`.
3. Run `claw config model` and `claw status --output-format json` in `/tmp` and verify only global settings load.
4. Run `claw prompt` in `/tmp` without `--model` and verify the global settings model is used.
5. Run a WebSearch prompt in `/tmp` without `--model` and verify the global provider can call the tool.
6. Run `claw config model` and `claw status --output-format json` in `/workspace/project-ok` and verify project settings override compatible provider details.
7. Run `claw prompt` and a WebSearch prompt in `/workspace/project-ok` without `--model`.
8. Run `claw config model` and `claw status --output-format json` in `/workspace` and verify the conflicting project model wins.
9. Run `claw prompt` in `/workspace` without `--model` with a timeout and verify it attempts the conflicting project provider instead of the global provider.
10. Inspect container mounts and `$CLAW_CONFIG_HOME`.
11. Confirm no `/home/claw/.codex`, `/home/claw/.config/claw`, or `/workspace/.claw` global mount is present.
## Expected Results
- The image builds from GitHub using `CLAW_REPO` and `CLAW_REF`.
- `/workspace` loads two files: `/home/claw/.claw/settings.json` and `/workspace/.claw/settings.json`.
- `/workspace` resolves `model` to `docker-project-model`.
- `/tmp` resolves `model` to the global model.
- Prompt execution without `--model` uses the same settings model shown by `config` and `status`.
- WebSearch works through settings-backed model/provider selection.
- A conflicting project `model` overrides global settings even when it points to a bad endpoint.
- Global state is mounted only at `/home/claw/.claw`.
## Logs To Capture
- Compose config excerpt.
- Build result.
- Config command stdout for global and project scopes.
- Mount/environment inspection output.
## Locator Contract
Not applicable; no browser UI is changed.
