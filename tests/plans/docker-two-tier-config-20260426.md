# Test Plan: docker-two-tier-config
## Created
2026-04-26
## Goal
Verify the `F:\desktop\docker\crawcode` Docker stack builds the local Claw source and preserves global/project `.claw` config precedence in mounted volumes.
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
2. Build the image with `docker compose build claw`.
3. Run `claw config model` in `/workspace` and verify the project model wins.
4. Run `claw config permissions.defaultMode` in `/workspace` and verify the project permission wins.
5. Run `claw config model` in `/tmp` and verify only global settings load.
6. Inspect container mounts and `$CLAW_CONFIG_HOME`.
7. Confirm no `/home/claw/.codex`, `/home/claw/.config/claw`, or `/workspace/.claw` global mount is present.
## Expected Results
- The image builds from GitHub using `CLAW_REPO` and `CLAW_REF`.
- `/workspace` loads two files: `/home/claw/.claw/settings.json` and `/workspace/.claw/settings.json`.
- `/workspace` resolves `model` to `docker-project-model`.
- `/tmp` resolves `model` to the global model.
- Global state is mounted only at `/home/claw/.claw`.
## Logs To Capture
- Compose config excerpt.
- Build result.
- Config command stdout for global and project scopes.
- Mount/environment inspection output.
## Locator Contract
Not applicable; no browser UI is changed.
