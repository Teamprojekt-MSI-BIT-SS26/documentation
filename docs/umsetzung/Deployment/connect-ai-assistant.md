# Connect an AI Coding Assistant

This page shows how to connect your AI coding assistant to the AI model on the GPU server. The model runs at IOS; your code and requests stay on this server and are not sent to an external provider. How the server is set up is described in the [AI Model Deployment guide](ai-model-deployment.md).

## What you need

- **A personal API key** (`sk-…`). Ask the admin of our team for one. The key is only for you: do not share it and never commit it. If it gets lost or leaked, tell the admin, who revokes it and gives you a new one.
- **Access from the HTWG network or the VPN.** From other networks, the server is not reachable.
- An assistant that can use an OpenAI-compatible API, e.g. [OpenCode](https://opencode.ai) or Cline.

## Connection data

| Setting | Value |
|---|---|
| Base URL | `https://ai-api.htwg-konstanz.de/v1` |
| API key | your personal key |
| Models | `coding-agent`, `coding`, `docs` |

`ai-api.htwg-konstanz.de` is a placeholder until the DNS name of the server is known.

Which model to use:

| Model | Use | Thinking |
|---|---|---|
| `coding-agent` | agentic programming: the assistant reads and edits files, runs commands and uses tools | on, maximum effort |
| `coding` | questions, explanations, small changes; answers faster | off |
| `docs` | documentation, READMEs and other longer texts | on, medium effort |

You can use up to 4 requests at the same time, so that everyone in the team gets a share of the GPUs.

## Check the connection

```bash
export HTWG_AI_API_KEY="sk-…"     # only for this terminal session
curl https://ai-api.htwg-konstanz.de/v1/models -H "Authorization: Bearer $HTWG_AI_API_KEY"
```

The answer lists the three models. If it does not, see [Problems](#problems).

## OpenCode

Store the key in a file that only you can read:

```bash
mkdir -p ~/.config/opencode
printf '%s' 'sk-…' > ~/.config/opencode/htwg-key
chmod 600 ~/.config/opencode/htwg-key
```

Add the provider to your global configuration `~/.config/opencode/opencode.json`, so that it works in every project:

```json
{
  "$schema": "https://opencode.ai/config.json",
  "model": "htwg/coding-agent",
  "small_model": "htwg/coding",
  "provider": {
    "htwg": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "HTWG",
      "options": {
        "baseURL": "https://ai-api.htwg-konstanz.de/v1",
        "apiKey": "{file:~/.config/opencode/htwg-key}"
      },
      "models": {
        "coding-agent": { "name": "Agentic programming", "reasoning": true, "tool_call": true, "limit": { "context": 131072, "output": 32000 } },
        "coding":       { "name": "Programming, fast", "tool_call": true, "limit": { "context": 131072, "output": 32000 } },
        "docs":         { "name": "Documentation", "reasoning": true, "tool_call": true, "limit": { "context": 131072, "output": 32000 } }
      }
    }
  }
}
```

Start `opencode` in a project and ask it, for example, to explain a file. Switch models with `/models`.

- **`limit`** tells OpenCode how much context the server allows. Do not change it; if the server changes, this page is updated.
- **Key as environment variable:** instead of the file, `"apiKey": "{env:HTWG_AI_API_KEY}"` with `export HTWG_AI_API_KEY="sk-…"` in `~/.zshrc` or `~/.bashrc` also works in principle. In a local test, OpenCode 2.2.7 did not send the key this way; if one variant fails, try the other.
- OpenCode also reads `opencode.json` files in the project and its parent directories. If it shows a different model than expected, check whether a project configuration overrides yours.

## Cline

In the settings of Cline:

| Field | Value |
|---|---|
| API Provider | OpenAI Compatible |
| Base URL | `https://ai-api.htwg-konstanz.de/v1` |
| API Key | your personal key |
| Model ID | `coding-agent` |

Other assistants are set up the same way: provider "OpenAI compatible", base URL, key and one of the three model names.

## Tips

- **Keep the beginning of the prompt the same.** The server reuses the beginning of earlier requests (system prompt, files read so far), which makes agents much faster. Avoid changing rules files, the system prompt or the set of tools during a session.
- **Use `coding` for quick questions.** It answers without thinking and is faster than `coding-agent`.
- **Check the results.** The model is good, but it is not on the level of the best commercial models; review changes before you commit them.

## Problems

| Problem | Solution |
|---|---|
| No connection, timeout | you are outside the HTWG network: connect to the VPN |
| `401 Unauthorized` | key mistyped, missing or revoked; check the key or ask the admin for a new one |
| `404 Not Found` | the base URL must end with `/v1` |
| `400` with "model not allowed" | the model name is wrong, or your key is not allowed to use it; use `coding-agent`, `coding` or `docs` |
| OpenCode: `Authentication Error, No api key passed in.` | switch between `{file:…}` and `{env:…}` for `apiKey`, see [OpenCode](#opencode) |
| OpenCode: `max_completion_tokens … cannot be greater than max_model_len` | `limit` is missing or too high in `opencode.json`; use the values from this page |
| Answers are slow | many people are working at the same time, or the prompt is very long; try again later or use `coding` |

For other problems, contact the admin of our team.
