# Tools

This page describes the tools we can work with and how they differ. For an overview of all building blocks, see [Overview](1-overview.md).

## Modes

Tools offer up to three modes. Which ones we need follows from the [survey](7-requirements.md#survey) (question 1).

| Mode | What it does |
|---|---|
| **Agent** | Works on a task on its own: reads and edits files, runs commands, checks the result and repeats until the task is done. |
| **Chat** | Answers questions and suggests code; we copy or apply the changes ourselves. |
| **Autocomplete** | Suggests the next lines while we type. Only possible in tools that run inside the editor. |

## Terminal tools

Terminal tools run in the terminal, next to any editor. They are built for agent mode and offer no autocomplete.

| Tool | Models | Notes |
|---|---|---|
| [OpenCode](https://opencode.ai) | Almost any provider and local engines | Each agent can use its own model, see [Hybrid](3-operating-models.md#hybrid). |
| [Claude Code](https://docs.claude.com/en/docs/claude-code/overview) | Anthropic models | Other models only through a gateway with an Anthropic-compatible API ([LLM gateway](https://code.claude.com/docs/en/llm-gateway)), e.g. the engine Strata. Also available as a VS Code extension. |
| [Codex CLI](https://github.com/openai/codex) | OpenAI models | Local models with `codex --oss` through Ollama or LM Studio ([Ollama docs](https://docs.ollama.com/integrations/codex)). |
| [Gemini CLI](https://github.com/google-gemini/gemini-cli) | Google Gemini models | Free with a personal Google account, limited to 1,000 requests per day ([quotas](https://geminicli.com/docs/resources/quota-and-pricing/)). |
| [Aider](https://aider.chat) | Almost any provider and local engines | Works closely with Git and commits every change. |

## VS Code extensions

VS Code extensions work inside the editor and show changes as diffs there.

| Tool | Models | Notes |
|---|---|---|
| [GitHub Copilot](https://github.com/features/copilot) | Models offered by GitHub | Agent, chat and autocomplete. Free for verified students through [GitHub Education](https://education.github.com), but with a reduced choice of models since March 2026; new student sign-ups were paused in April 2026 ([GitHub docs](https://docs.github.com/en/copilot/how-tos/copilot-on-github/set-up-copilot/enable-copilot/set-up-for-students)). |
| [Cline](https://cline.bot) | Almost any provider and local engines | Plan and Act mode can use different models ([Cline docs](https://docs.cline.bot/core-workflows/plan-and-act)). |
| [Roo Code](https://roocode.com) | Almost any provider and local engines | Each mode remembers its own model ([Roo Code docs](https://docs.roocode.com/basic-usage/using-modes)). |
| [Kilo Code](https://kilocode.ai) | Almost any provider and local engines | Still to be checked. |

## AI editors

AI editors are separate editors based on VS Code with agent, chat and autocomplete built in. They are mostly used with the provider's subscription.

- [Cursor](https://cursor.com)
- [Windsurf](https://windsurf.com)

## No longer maintained

- **Continue:** was taken over by Cursor and is no longer developed. The code stays available as open source.

## What to compare

For each tool we look at:

- Which modes it offers (agent, chat, autocomplete)
- Which models and providers it can use, including local engines
- Costs, free tiers and student offers
- Limits such as requests per day
- Data privacy: whether our code is used for training
- Setup effort in VS Code
