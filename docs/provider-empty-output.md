# Provider Empty Output Recovery

OpenAI-compatible local providers can occasionally close a stream with a final stop event but no text block and no tool call. Claw treats this as a provider-side empty output, not as a valid assistant turn.

| Goal | File |
|---|---|
| Runtime recovery | `rust/crates/runtime/src/conversation.rs` |
| CLI error kind | `rust/crates/rusty-claude-cli/src/main.rs` |
| Test plan | `tests/plans/provider-empty-output-retry-20260426.md` |
| Test reports | `tests/reports/provider-empty-output-retry/` |

## Contract

- A stopped stream with no assistant text and no tool call is invalid.
- The runtime retries the same request once before failing the turn.
- The retry must not duplicate the user message or persist an empty assistant message.
- If the retry is also empty, the user-visible error kind is `provider_empty_output`.
- This recovery is intentionally local to empty provider output. API HTTP failures, context-window failures, and malformed non-empty streams keep their existing error paths.
