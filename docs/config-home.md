# Config Home

Claw Code uses a two-tier `.claw` policy for runtime state and project overrides.

## Active Tiers

| Tier | Path | Purpose |
|------|------|---------|
| Global | `$CLAW_CONFIG_HOME/settings.json` or `$HOME/.claw/settings.json` | Providers, model registry, installed skills, slash commands, agents, plugins, sessions, user defaults |
| Project | `<project>/.claw/settings.json` | Project-specific overrides |

The runtime config loader does not load `.claw.json`, `.claude`, `.codex`, `.config/claw`, or `.claw/settings.local.json`.

## Runtime State

Managed sessions are written under:

```text
$CLAW_CONFIG_HOME/sessions/<workspace_hash>/
```

The workspace hash keeps sessions separated per project while keeping storage in the global Claw home.

## Docker Contract

For `F:\desktop\docker\crawcode`, the container should mount:

```yaml
volumes:
  - ./workspace:/workspace
  - ./claw-home/.claw:/home/claw/.claw
environment:
  CLAW_CONFIG_HOME: /home/claw/.claw
```

`./workspace/.claw/settings.json` is the only project config file path. `./workspace/.claw/{skills,commands,agents}` owns project definitions. `./claw-home/.claw/` owns global settings, definitions, and runtime state.

The local Dockerfile is expected to clone `CLAW_REPO` at `CLAW_REF`. During feature validation, set `CLAW_REF` to a pushed feature branch; after merging, set it back to `main`.

## Model Precedence

The merged settings `model` is used by both REPL and one-shot prompt execution when `--model` is not supplied.

Precedence:

1. `--model`
2. `<project>/.claw/settings.json`
3. `$CLAW_CONFIG_HOME/settings.json`
4. built-in default model

This applies to `claw prompt`, `claw -p`, shorthand prompt mode, piped prompts, and slash-command paths that invoke a prompt.

The REPL `Connected: <model> via <provider>` line reports the provider kind from the constructed runtime client. For models declared in `models`, this means the label follows the linked `providers.<name>.type` value instead of guessing from the model name.

## Definition Cascade

Skills, slash-command markdown, and agents use the same two tiers:

1. project `.claw`
2. `$CLAW_CONFIG_HOME`

Lookup paths:

- Skills: `<project>/.claw/skills`, `$CLAW_CONFIG_HOME/skills`
- Slash-command markdown: `<project>/.claw/commands`, `$CLAW_CONFIG_HOME/commands`
- Agents: `<project>/.claw/agents`, `$CLAW_CONFIG_HOME/agents`

For duplicate skill, slash-command, or agent names, project definitions stay active and global definitions are reported as shadowed.

Claude-style rule directories such as `.claude/rules` are not implemented in this runtime and are outside the current two-tier definition cascade.
