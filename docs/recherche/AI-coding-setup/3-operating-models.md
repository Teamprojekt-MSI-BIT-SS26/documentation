# Operating models

The operating model decides where the model runs, who pays for it and which parts we have to set up ourselves.

## Local

The model runs on our own laptops: each team member runs it on their own machine with an engine such as [Ollama](https://ollama.com), [LM Studio](https://lmstudio.ai), [llama.cpp](https://github.com/ggml-org/llama.cpp), [MLX](https://github.com/ml-explore/mlx) or [Strata](https://github.com/niko1221/strata). The tool connects to the engine on `localhost`.

- **Pro:** no running costs; works offline; code never leaves the laptop.
- **Contra:** quality and speed are limited by the hardware; agent mode needs large models and a lot of RAM; each person sets up and maintains their own installation.
- **To check:** which models run at a usable speed on the laptops from the survey (question 6).

## Own server

One machine with a strong GPU runs the model for the whole team, e.g. a rented GPU server or a server at HTWG. A server engine such as [vLLM](https://docs.vllm.ai) serves several requests at the same time.

- **Pro:** larger models than on laptops; one setup for everyone; data stays under our control.
- **Contra:** rental costs per hour or month; someone has to run and secure the server; access from outside must be protected.
- **To check:** whether HTWG offers GPU servers or its own AI services; prices for GPU rental.

## Pay per token (API)

We pay per request, measured in tokens. A tool such as [Cline](https://cline.bot), [Roo Code](https://roocode.com) or [OpenCode](https://opencode.ai) connects to the API of a provider or of a router such as [OpenRouter](https://openrouter.ai), which offers many proprietary and open-weight models through one interface.

- **Pro:** pay only for actual usage; free choice of model; cheap open-weight models are available.
- **Contra:** costs are hard to predict, because agents send long contexts and use many tokens; one API key needs to be managed per person or for the team.
- **To check:** price per million tokens of suitable models; whether providers use our data for training.

## Subscription

A fixed monthly price, usually with a VS Code extension included. Examples: [GitHub Copilot](https://github.com/features/copilot), [Cursor](https://cursor.com), [Claude Pro](https://claude.com/pricing).

- **Pro:** easy to set up, predictable costs, usually strong models.
- **Contra:** usage limits; costs per person; code is sent to the provider.
- **To check:** student offers (e.g. [GitHub Education](https://education.github.com)) and free tiers.

## Hybrid

Combinations of the options above, for example:

- a small local model for autocomplete and a paid API only for agent tasks
- a paid tool only for heavy users and a free tool for everyone else (see [requirements](7-requirements.md), decision 3)
- a strong cloud model for writing code and a cheap or local model for searching and reading code

### Switching between models

Tools like OpenCode do not choose a model per request on their own. Instead, each agent is assigned a model once, and the tool sticks to it ([OpenCode agents](https://opencode.ai/docs/agents/)):

- **Primary agents** such as *Build* and *Plan* are the agents we talk to. Each can have its own model; we switch between them with the **Tab** key.
- **Subagents** such as *Explore* are called by the primary agent automatically when a subtask matches their description. They can have their own model too. Without one, they use the model of the agent that called them.

A hybrid setup in `opencode.json` could look like this (model names are placeholders):

```json
{
  "agent": {
    "build":   { "model": "<cloud-provider>/<strong-model>" },
    "plan":    { "model": "<cloud-provider>/<cheap-model>" },
    "explore": { "model": "ollama/<local-model>" }
  }
}
```

The expensive model writes the code, while searching and reading the code base, which uses many tokens, runs on a cheap or local model automatically.

### What to watch out for

- **Data privacy per model:** If one agent uses a cloud model, its context goes to the cloud. A setup is only as private as the setting of each agent.
- **Tool calling of small models:** Subagents have to call tools reliably. Small local models often fail at this, so the shared test task should cover it.
- **Context:** Subagents start with a fresh context and only return a summary. Switching the model in the middle of a session breaks the provider's prompt cache, so the whole history is processed and paid for again.
- **Setup:** two providers or a provider plus a local engine. A shared `opencode.json` in the repository keeps the setup the same for everyone; API keys never go into this file.
- **Costs:** tokens are spent in several places. A budget limit at the API provider helps to keep control.

## Comparison

| | Local | Own server | Pay per token | Subscription |
|---|---|---|---|---|
| **Running costs** | none | fixed for the server | depends on usage | fixed per person |
| **Setup effort** | medium | high | low to medium | low |
| **Maintenance** | per person | one person for the team | low | none |
| **Agent mode** | only with large models and much RAM | yes, depending on model and GPU | yes, depending on model | yes |
| **Limits** | hardware speed | GPU capacity | budget | usage limits of the plan |
| **Data privacy** | stays on the laptop | stays on our server | code goes to the provider | code goes to the provider |
