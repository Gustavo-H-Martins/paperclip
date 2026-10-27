---
title: Anthropic-Compatible Endpoints
summary: Run claude_local against a third-party API that implements the Anthropic Messages API
---

The `claude_local` adapter runs the Claude Code CLI, and the CLI accepts any
endpoint that implements the Anthropic Messages API. Two environment variables
are enough to move an agent off the Anthropic API: `ANTHROPIC_BASE_URL` and
`ANTHROPIC_AUTH_TOKEN`. Providers that expose an Anthropic-compatible route (for
example DeepSeek at `https://api.deepseek.com/anthropic`) work this way without a
separate adapter.

## How the values reach the CLI

`claude_local` starts `claude` with the server process environment merged under
the agent's adapter `env`, so the pair can be set in either place:

- **Server environment.** Export the variables before starting Paperclip, and
  every `claude_local` agent inherits them. This is the shortest path when the
  whole instance uses one provider.
- **Adapter `env`.** Set them per agent, which is what you want when only some
  agents leave the Anthropic API. Secret references are supported, so the token
  does not sit in plain text in the agent configuration.

Both execution lanes honor the pair. On the ACP lane the environment probe seeds
a host `ANTHROPIC_AUTH_TOKEN` into the probe environment, so the "Test
Environment" button does not report a missing credential for a token-only setup.

| Variable | Role |
|----------|------|
| `ANTHROPIC_BASE_URL` | Root URL of the compatible endpoint. The CLI appends `/v1/messages` to it. |
| `ANTHROPIC_AUTH_TOKEN` | Bearer token sent to that endpoint. Leave `ANTHROPIC_API_KEY` unset so the CLI does not prefer API-key auth. |
| `ANTHROPIC_MODEL` | Default model id the CLI sends when the agent config sets no `model`. |
| `ANTHROPIC_SMALL_FAST_MODEL` | Model id for the CLI's small background calls. Set it when the provider rejects the built-in small model. |

## Model ids

The adapter and the model picker describe Anthropic models, so the defaults do
not apply to a third-party endpoint:

- An unset `model` resolves to `claude-opus-5`, and a compatible endpoint
  usually rejects that id. Set the adapter `model` field to a model id the
  provider advertises. An explicit `model` wins over `ANTHROPIC_MODEL`.
- The picker's live model list is fetched with `ANTHROPIC_API_KEY`. With a
  token-only setup it falls back to the built-in Claude list, so type the
  provider's id rather than picking one from the list.

## Costs and budgets

Usage is still recorded per run, because the CLI reports tokens in its output.
Two accounting limits apply to endpoints outside the Anthropic API:

- The billing identity is read from `ANTHROPIC_API_KEY` and the Bedrock
  variables. With only `ANTHROPIC_AUTH_TOKEN` set, runs are classified as
  `subscription` rather than metered API usage.
- Paperclip has no price list for the provider, so the Costs view reports those
  entries as unpriced rather than at a dollar amount, and spend-based budget
  enforcement cannot see the traffic.

Enforce the spend cap on the provider's own key or account, and treat the
Paperclip cost view for these agents as a token counter.

## Setup steps

1. Configure the endpoint and token, either in the server environment or in the
   agent's `env`.
2. Set the agent `model` to an id the provider serves.
3. Open the agent, run "Test Environment", and confirm the hello probe passes.
4. Run a heartbeat and read the issue thread to confirm the response came from
   the expected model.

Agent configuration example, with the token stored as a secret reference:

```json
{
  "adapterType": "claude_local",
  "adapterConfig": {
    "model": "<provider model id>",
    "env": {
      "ANTHROPIC_BASE_URL": "https://api.deepseek.com/anthropic",
      "ANTHROPIC_AUTH_TOKEN": {
        "type": "secret_ref",
        "secretId": "<secret id>",
        "version": "latest"
      }
    }
  }
}
```

To start the server with the endpoint applied to every agent on a local install:

```sh
export ANTHROPIC_BASE_URL="https://api.deepseek.com/anthropic"
export ANTHROPIC_AUTH_TOKEN="<token>"
paperclipai onboard --yes
```

## When not to use this

If the provider speaks the OpenAI API rather than the Anthropic Messages API, use
the Codex or OpenCode adapter instead. If the agent needs Anthropic tool-use
behavior that the compatible endpoint does not implement, the run fails at the
first tool call rather than at startup, so test with a task that uses a tool
before scheduling heartbeats.
