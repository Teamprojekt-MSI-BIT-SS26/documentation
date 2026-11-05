# Tools

This page describes the tools we can work with and how they differ.

## Modes

Tools offer up to three modes.

| Mode | What it does |
|---|---|
| **Agent** | Works on a task on its own: reads and edits files, runs commands, checks the result and repeats until the task is done. |
| **Chat** | Answers questions and suggests code; we copy or apply the changes ourselves. |
| **Autocomplete** | Suggests the next lines while we type. Only possible in tools that run inside the editor. |

## Why the tool matters

The same model can deliver different results in different tools. Parts of the tool can change benchmark scores by about as much as the difference between two model generations ([Coding benchmarks are misaligned with agentic software engineering](https://arxiv.org/abs/2606.17799)).

| What the tool does | Influence on the result |
|---|---|
| **Choosing the context** | Which files and parts of the code are sent to the model, e.g. through a map of the repository ([Aider repo map](https://aider.chat/docs/repomap.html)) or a search in the code. Without the relevant code, even a strong model cannot solve the task. |
| **Instructions and actions** | Which instructions the model gets and which actions it may use: read and edit files, run commands, start tests. |
| **Format of the changes** | Whether the model returns whole files, diffs or search-and-replace blocks. Different models work better or worse with different formats ([Aider edit formats](https://aider.chat/docs/more/edit-formats.html)). |
| **Workflow** | Whether the tool plans, checks the result with tests or a linter and fixes errors itself, or stops after one answer. |
| **Tuning to models** | Some tools are built for the models of one provider, e.g. Claude Code for Anthropic's models. |

Benchmark scores are therefore only valid for the combination of model and tool they were measured with.

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
| [GitHub Copilot](https://github.com/features/copilot) | Models offered by GitHub | Agent, chat and autocomplete. Free for verified students through [GitHub Education](https://education.github.com); on the student plan, models are only available through automatic selection ([GitHub docs](https://docs.github.com/en/copilot/concepts/billing/individual-plans)). |
| [Cline](https://cline.bot) | Almost any provider and local engines | Plan and Act mode can use different models ([Cline docs](https://docs.cline.bot/core-workflows/plan-and-act)). |
| [Kilo Code](https://kilocode.ai) | Almost any provider and local engines | Still to be checked. |

## AI editors

AI editors are separate editors based on VS Code with agent, chat and autocomplete built in. They are mostly used with the provider's subscription.

| Tool | Models | Notes |
|---|---|---|
| [Cursor](https://cursor.com) | Models offered by Cursor | Agent, chat and autocomplete. |
| [Devin Desktop](https://devin.ai/desktop) | Models offered by Devin Desktop | Agent, chat and autocomplete. |
