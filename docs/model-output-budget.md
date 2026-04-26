# Model Output Budget

Claw controls response length from settings, not provider-specific environment variables.

| Goal | File |
|---|---|
| Usage contract | `USAGE.md` |
| Context diagnostic | `docs/context-command.md` |
| Runtime config parser | `rust/crates/runtime/src/config.rs` |
| CLI request budget | `rust/crates/rusty-claude-cli/src/main.rs` |
| Test plan | `tests/plans/model-output-budget-20260426.md` |
| Test reports | `tests/reports/model-output-budget/` |

## Contract

- `models[].maxContext` is the total model context window.
- `models[].maxOutputTokens` is an optional response cap for the selected model.
- Each request sends `max_tokens = min(configured output cap, maxContext - estimated input - 4096)`, with a 256-token floor.
- The system prompt includes the active response budget so the model can choose a compact answer shape before hitting the hard provider cap.
- Provider stop reasons `length`, `max_tokens`, and `max_output_tokens` mark the turn as output-limited.
- `/context` reports both the configured output cap and the next request output budget.

This does not yet summarize oversized successful assistant messages before the next turn. That is a separate conversation-memory policy.
