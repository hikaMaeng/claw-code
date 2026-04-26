# Context Command

`/context` is a local context-window diagnostic.

| Goal | File |
|---|---|
| Usage contract | `USAGE.md` |
| Rust command surface | `rust/crates/rusty-claude-cli/src/main.rs` |
| Test plan | `tests/plans/context-command-20260426.md` |
| Test reports | `tests/reports/context-command/` |

## Contract

- `/context`, `/context show`, and `/context usage` are read-only.
- `claw context`, `claw /context`, and resumed `claw --resume SESSION.jsonl /context` are local commands.
- The command must not issue an LLM request.
- `models[].maxContext` is the context-window source when the selected model is configured.
- `models[].maxOutputTokens` is the configured response cap when present.
- The auto-compact line uses the same threshold as runtime auto-compaction.
- Text and JSON output include message estimate, system prompt estimate where available, autocompact buffer, free space, next request output budget, config-file counts, and memory-file count.
- Text output is intentionally compact: one usage headline, one ASCII context bar, a short category table, and a footer with workspace/config/session hints.
- `/context clear` is rejected; users must use `/clear` or `/compact`.
