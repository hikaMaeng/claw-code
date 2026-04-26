# Claude Feature Coverage

This note records which Claude Code-style feature surfaces are implemented in the Rust runtime.

## Summary

| Feature | Status | Runtime source |
|---------|--------|----------------|
| Two-tier settings | Implemented | `$CLAW_CONFIG_HOME/settings.json`, `<project>/.claw/settings.json` |
| Model registry | Implemented | `providers`, `models`, `aliases`, `model` in settings |
| Skills | Implemented | `<project>/.claw/skills`, `$CLAW_CONFIG_HOME/skills` |
| Slash command markdown | Partial | Loaded as legacy command skills from `.claw/commands`; not top-level custom slash dispatch |
| Agents | Implemented | `<project>/.claw/agents`, `$CLAW_CONFIG_HOME/agents` |
| Hooks | Partial | `PreToolUse`, `PostToolUse`, `PostToolUseFailure` only |
| Claude rule directories | Not implemented | No `.claude/rules`, `.claude/rule`, `.claw/rules`, or `.claw/rule` loader |
| Project memory | Partial | `CLAUDE.md`, `CLAUDE.local.md`, `.claw/CLAUDE.md`, `.claw/instructions.md` only |
| MCP servers | Implemented through settings | Merged by runtime config loader |
| Plugins | Partial | Tools/resources/hooks/lifecycle supported; Claude plugin skills, agents, MCP imports, and directory-glob commands rejected |
| Sessions | Implemented | Managed under `$CLAW_CONFIG_HOME/sessions/<workspace_hash>/` |

## Skills

Skills use the two-tier cascade:

1. `<project>/.claw/skills`
2. `$CLAW_CONFIG_HOME/skills`

Duplicate names keep the project definition active and report the global definition as shadowed. The relevant discovery code is in `rust/crates/commands/src/lib.rs` under `discover_skill_roots`.

## Slash Commands

Built-in slash commands are registered statically in `rust/crates/commands/src/lib.rs`.

Markdown files under `.claw/commands` are supported only as legacy command skills. They are resolved through the same `/skills` path as skills, not as arbitrary top-level `/name` slash commands.

Plugin manifests that use Claude Code-style command directory globs are explicitly rejected with a validation error.

## Agents

Agents use the two-tier cascade:

1. `<project>/.claw/agents`
2. `$CLAW_CONFIG_HOME/agents`

Duplicate names follow the same project-over-global precedence as skills. The relevant discovery code is `discover_definition_roots(cwd, "agents")`.

## Hooks

Runtime settings and plugin manifests currently support exactly these hook events:

- `PreToolUse`
- `PostToolUse`
- `PostToolUseFailure`

Claude Code lifecycle hooks such as `SessionStart` are not accepted by plugin manifest validation and are not represented in `runtime::hooks::HookEvent`.

## Rules

Claude-style rule directories are not implemented.

The code search found no loader for these paths:

- `.claude/rules`
- `.claude/rule`
- `.claw/rules`
- `.claw/rule`

Existing `rules` references in the Rust runtime are permission rules or test fixture text, not Claude rule-file support. The implemented permission settings are `permissions.allow`, `permissions.deny`, `permissions.ask`, and `permissions.defaultMode`.

## Memory Files

Prompt context loads instruction files from the current directory and its ancestors. The only file names discovered by `rust/crates/runtime/src/prompt.rs` are:

- `CLAUDE.md`
- `CLAUDE.local.md`
- `.claw/CLAUDE.md`
- `.claw/instructions.md`

There is no global `$CLAW_CONFIG_HOME` memory-file loader and no rule-directory loader in the prompt context path.

## Plugin Contract Gaps

The plugin crate rejects several Claude Code plugin contract fields:

- `skills`
- `agents`
- `mcpServers`
- string-based command directory globs in `commands`
- hook names other than `PreToolUse`, `PostToolUse`, `PostToolUseFailure`

This means plugin-provided skills, agents, MCP servers, and slash command markdown catalogs are not currently part of the runtime surface.
