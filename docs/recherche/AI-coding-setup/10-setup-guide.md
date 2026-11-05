# Setup guide

This guide sets up OpenCode with the reference model Qwen3.6-35B-A3B through GWDG Chat AI, option 2 from the [Evaluation](9-evaluation.md#result). It follows the official GWDG guide for [agentic coding](https://docs.hpc.gwdg.de/services/ai-services/coco-ai/agentic_coding/index.html). If the HTWG server becomes available later, only the address of the API changes.

## 1. Account and API key

1. Sign in at [Academic Cloud](https://academiccloud.de) with the HTWG account through the federated login. If HTWG is not offered, the option cannot be used; please tell the team.
2. Request an API key on the [KISSKI LLM service page](https://kisski.gwdg.de/en/leistungen/2-02-llm-service/) with "Book". Use the same email address as in the Academic Cloud account ([SAIA](https://docs.hpc.gwdg.de/services/ai-services/saia/index.html)).
3. Keep the key to yourself. Every team member needs their own key; the limits apply per key: 10 requests per minute, 200 per hour, 400 per day, 3,000 per month.

## 2. Store the API key

Set the key as an environment variable, so it never ends up in a file of the repository.

=== "macOS (zsh)"

    ```bash
    echo 'export SAIA_API_KEY="your-saia-api-key"' >> ~/.zshrc
    source ~/.zshrc
    ```

=== "Windows (PowerShell)"

    ```powershell
    setx SAIA_API_KEY "your-saia-api-key"
    ```

    Open a new terminal afterwards.

Check that the key works and that the model is available:

```bash
curl -s https://chat-ai.academiccloud.de/v1/models \
  -H "Authorization: Bearer $SAIA_API_KEY" | grep qwen3.6-35b-a3b
```

## 3. Install OpenCode

```bash
npm install -g opencode-ai
opencode --version
```

Alternatively: `curl -fsSL https://opencode.ai/install | bash` on macOS and Linux.

## 4. Configuration

Every team member creates `opencode.json` in the root of their project and does not commit it. The file contains no secret, only the name of the environment variable.

```json
{
  "$schema": "https://opencode.ai/config.json",
  "share": "disabled",
  "provider": {
    "saia": {
      "npm": "@ai-sdk/openai-compatible",
      "name": "GWDG SAIA",
      "options": {
        "baseURL": "https://chat-ai.academiccloud.de/v1",
        "apiKey": "{env:SAIA_API_KEY}"
      },
      "models": {
        "qwen3.6-35b-a3b": {
          "name": "Qwen3.6-35B-A3B (GWDG)"
        },
        "qwen3-coder-next": {
          "name": "Qwen3-Coder-Next (GWDG)"
        }
      }
    }
  },
  "model": "saia/qwen3.6-35b-a3b"
}
```

- `model` sets the reference model as default. Qwen3-Coder-Next is available as a second model and can be chosen with `/models`.
- The model names must match the [list of the SAIA API](https://docs.hpc.gwdg.de/services/ai-services/saia/index.html) exactly.
- `"share": "disabled"` prevents sessions from being shared through a public link ([OpenCode config schema](https://opencode.ai/config.json)).
- Tested with OpenCode 2.0.29. In this version, the `provider` block is marked as deprecated but still works; the GWDG guide uses it as well.

## 5. Use it in VS Code

1. Open the Extensions view (`Cmd+Shift+X` on macOS, `Ctrl+Shift+X` on Windows), search for **OpenCode** and install it.
2. Reload VS Code, so that it picks up `SAIA_API_KEY`.
3. Open the OpenCode panel and start with a small task.

The extension uses the same `opencode.json`. OpenCode also runs in the terminal with `opencode`.

| Command | Action |
|---|---|
| `/models` | Switch the model |
| `Tab` | Switch between the agents Plan (only plans) and Build (edits files) ([OpenCode](https://opencode.ai/docs/agents/)) |
| `@` | Add a file as context |

If the models do not show up, `SAIA_API_KEY` is not loaded: open a new terminal or restart VS Code.

If OpenCode reports `No authentication provided`, its background server was started before the key was set. OpenCode 2 runs this server in the background and uses it for `opencode` and the extension. Restart it from a terminal that knows the key:

```bash
source ~/.zshrc && opencode service restart
```

Closing OpenCode with `Ctrl+C` does not stop a running session: the background server continues it and keeps sending requests. To stop a session for good, restart the background server with the same command.

## 6. Data privacy

- **Data stays at GWDG:** the internal models run on GWDG's infrastructure; messages are not stored on the server and not used for training ([Chat AI FAQ](https://docs.hpc.gwdg.de/services/ai-services/chat-ai/faq/index.html)).
- **No external models:** the models of OpenAI and Anthropic marked "(External)" in Chat AI are not available through the API anyway; do not use them for project code in the web interface.
- **No real student data:** do not enter real data of students from Moodle in any tool, also not in prompts or test files.
- **Secrets:** OpenCode reads files in the project, so keep `.env` files and keys out of the project or out of the agent's reach.

## 7. Check

The setup works if OpenCode answers in VS Code with the model `saia/qwen3.6-35b-a3b` and can edit a file in the project. Afterwards, note the setup effort and the result of the test task in the [Evaluation](9-evaluation.md#4-comparison-of-the-options).
