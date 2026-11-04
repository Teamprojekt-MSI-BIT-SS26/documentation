# Evaluation

## Result

**Recommendation:** an open-weight model, Qwen3.6-35B-A3B, with OpenCode or Cline instead of the Claude subscription. The model can be used in three ways; the tool stays the same, only the address of the API changes.

| # | Option | Costs | Assessment |
|---|---|---|---|
| 1 | **HTWG server** (IOS) | none | Best option, if we get access: no limits, our code stays at HTWG. Access is not clarified yet. |
| 2 | **GWDG Chat AI** (Academic Cloud) | none | Same model, free, data is not stored. Quick to try, but limited to 400 requests per day and 3,000 per month per API key, possibly too little for agent mode. |
| 3 | **Usage-based API** (OpenRouter) | about 10–20 $ per person and month (estimate) | Same model without limits, training can be excluded. Probably cheaper than the subscription, but the estimate is still uncertain. |
| – | GitHub Copilot Student | none | Not recommended for agent mode: 200 AI credits per month, model chosen automatically. Not tested yet. |
| – | Gemini CLI | – | Dropped: the free tier was replaced by Antigravity CLI in June 2026. |

- **Quality:** Qwen3.6-35B-A3B is slightly below Claude Sonnet 4.5 from September 2025. The Claude models of our current subscription are newer, so we expect a noticeable drop in quality ([step 4](#4-comparison-of-the-options)).
- **Laptops:** not suitable for agent mode; small models are at most an addition for simple questions ([Our resources](8-our-resources.md#summary)).

**Next steps:**

1. Ask IOS whether and how we can use the server.
2. Check whether HTWG accounts can log in to Academic Cloud, and test GWDG Chat AI with OpenCode on a real task.
3. Read the token usage of team members who program with [ccusage](https://github.com/ryoppippi/ccusage) to confirm the cost estimate in [step 4](#4-comparison-of-the-options).
4. Test the chosen option briefly in practice before the whole team switches.

## Approach

Instead of testing every option ourselves, this evaluation is based on published sources and our own calculations. Each step builds on the previous one. The type of source is marked:

- **Study:** scientific publication
- **Vendor:** documentation or model card of the developer
- **Community:** reviews and recommendations of users
- **Calculation:** our own calculation from the sources above

## 1. Agent mode needs at least 64K context

Agent mode is a must-have ([requirements](7-requirements.md#requirements-list)). Agents read whole files and keep the history of all steps, so they need a large context. Ollama recommends at least 64K tokens for agents and coding tools. *(Vendor: [Ollama](https://docs.ollama.com/context-length))*

A larger context is not automatically better: models use information in the middle of a long context worse. *(Study: [Lost in the Middle](https://arxiv.org/abs/2307.03172))*

## 2. A capable model

For agent mode, a model has to call tools reliably and solve real programming tasks. A common measure is **SWE-bench Verified**: the share of real GitHub issues a model resolves. *(Study: [SWE-bench](https://arxiv.org/abs/2310.06770))*

??? info "How SWE-bench works"
    The model gets the code of a real open-source Python project, e.g. Django, and the description of an issue that was already solved on GitHub. Like an agent, it has to find the cause, read and edit files and deliver a fix. The task counts as solved if the tests belonging to the original fix now pass and no other tests break.

    **Verified** is a set of 500 of these tasks that were checked by humans to be clearly described and solvable. The value is the share of solved tasks: 73.4 % means about 367 of 500.

    SWE-bench is close to agent mode, because the model has to find its way through an unknown project. It only covers Python, and the tasks are public, so a model may have seen them during training.

Open-weight models recommended by the community for local agentic coding *(Community: [InsiderLLM](https://insiderllm.com/guides/best-local-coding-models-2026/), [Latent Space](https://latent.space/p/ainews-top-local-models-list-april/))*:

| Model | Type | SWE-bench Verified | Source of the value |
|---|---|---|---|
| Qwen3.6-27B | dense, 27B | 77.2 % | [Model card](https://huggingface.co/Qwen/Qwen3.6-27B) |
| Qwen3.6-35B-A3B | mixture of experts, 3B active | 73.4 % | [Model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) |
| Qwen3-Coder-Next | mixture of experts, 80B, 3B active | 70.6 % | [Technical report](https://arxiv.org/abs/2603.00729) |
| Devstral Small 2 | dense, 24B | 68.0 % | [Model card](https://huggingface.co/mistralai/Devstral-Small-2-24B-Instruct-2512) |

**Threshold: about 70 % on SWE-bench Verified and native tool calling.** There is no established minimum score for agent mode, so this is a guide value set by us: the models the community recommends for agentic coding reach 68–77 %, models below are hardly recommended, mainly because their tool calling is unreliable *(Community: [InsiderLLM](https://insiderllm.com/guides/best-local-coding-models-2026/))*. For comparison, Anthropic reported 77.2 % for Claude Sonnet 4.5 in September 2025, with a similar setup of bash and file-editing tools *(Vendor: [Anthropic](https://www.anthropic.com/news/claude-sonnet-4-5))*.

**Reference model: Qwen3.6-35B-A3B.** It is above the threshold, has a context of 262K tokens and is licensed under Apache 2.0 *(Vendor: [model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B))*. It was chosen over the stronger Qwen3.6-27B for two reasons:

- **Speed:** per token, it reads only its 3B active parameters, the 27B dense model all 27B, so it answers several times faster on the same hardware (see [Active parameters](5-models.md#active-parameters)).
- **Availability:** it is offered by GWDG Chat AI and on OpenRouter, so all three options in the [result](#result) can use the same model.

!!! note "Limits of benchmarks"
    The values are reported by the developers. The result also depends on the tool around the model, so values of the same model can differ: Mistral reports 68.0 % for Devstral Small 2, an independent measurement on [llmrun.dev](https://llmrun.dev/benchmark/swe-bench-verified?show=all) 56.4 %. *(Study: [Coding benchmarks are misaligned with agentic software engineering](https://arxiv.org/abs/2606.17799))*

## 3. Our resources

Whether our laptops and the possible HTWG server can run the reference model is examined in [Our resources](8-our-resources.md):

- **Laptops:** not suitable for agent mode. The reference model needs about 25 GB with Q4_K_M and 64K context, the laptops have at most 16 GB. On the Macs, small models are enough for simple tasks ([Result for the laptops](8-our-resources.md#result-for-the-laptops)).
- **Server:** under the assumption of 2 × A100 40 GB, vLLM runs the reference model in FP8 with 64K context for the whole team ([Result for the server](8-our-resources.md#result-for-the-server)).

## 4. Comparison of the options

The current Claude subscription is included as the reference.

| | Claude (current) | HTWG server | GWDG Chat AI | Usage-based API | GitHub Copilot Student |
|---|---|---|---|---|---|
| **Operating model** | [Subscription](3-operating-models.md#subscription) | [Own server](3-operating-models.md#own-server) | Service of an academic data centre, free API | [Usage-based API](3-operating-models.md#usage-based-api), e.g. [OpenRouter](https://openrouter.ai/models) | Student offer |
| **Model** | Claude models | Qwen3.6-35B-A3B | Qwen3.6-35B-A3B | Qwen3.6-35B-A3B | chosen automatically |
| **Quality** | strongest; Anthropic no longer reports SWE-bench Verified | 73.4 % on SWE-bench Verified | as HTWG server | as HTWG server | depends on the chosen model |
| **Tool** | Claude Code in the terminal or as a VS Code extension | OpenCode in the terminal or Cline in VS Code | OpenCode or Cline | OpenCode or Cline | Copilot in VS Code |
| **Features compared to Claude** | reference: agent and chat | agent and chat as with Claude Code; model and tool can be chosen freely | as HTWG server | as HTWG server | agent and chat, additionally autocomplete; model cannot be chosen |
| **Costs** | about 24.50 € per person per month, about 196 € for the team | none for us, if HTWG provides the server | free | about 0.05–0.25 $ per million input tokens, 0.025–0.25 $ per million cached tokens and 0.62–1.80 $ per million output tokens, depending on the provider | free for verified students |
| **Setup effort** | none, in use | high: engine, access from outside and shared use have to be set up once; low for each person afterwards | low: Academic Cloud login, book an API key, enter the endpoint in the tool | low: API key and an entry for the provider in the tool's configuration | low: install the extension, verify student status |
| **Limits** | five-hour session limit and weekly limit | capacity of the two GPUs | per API key 10 requests per minute, 200 per hour, 400 per day, 3,000 per month | costs | 200 AI credits per month |
| **Our code used for training** | only if the setting "Model improvement" is on | no, it stays at HTWG | no; messages are not stored on the server | can be excluded (`data_collection: deny`, zero data retention) | yes by default since 24 April 2026, opt-out in the settings |
| **Open points** | | access, own engine, shared use | whether HTWG accounts can log in | cost estimate based on one documentation session | not tested yet |

**Quality:** the developers report 73.4 % on SWE-bench Verified for Qwen3.6-35B-A3B and 77.2 % for Claude Sonnet 4.5 (September 2025), both with bash and file-editing tools. The reference model is therefore slightly below Sonnet 4.5. The current Claude models are newer; Anthropic compares them with other benchmarks, so there is no direct value. We therefore expect a noticeable drop in quality compared to today. How large it is for our work can only be shown by a practical test.

**GitHub Copilot Student:** since June 2026, the plan only has 200 AI credits per month and automatic model selection. Students report that the credits were used up after about 10 requests in agent mode; chat then stops until the next month, only autocomplete stays unlimited *(Community: [GitHub discussion](https://github.com/orgs/community/discussions/197557))*. We have not tested this ourselves yet.

**Cost estimate for the usage-based API:** Claude Code stores the sessions locally for 30 days *(Vendor: [Claude Code data usage](https://code.claude.com/docs/en/data-usage.md#data-retention))*, so the usage can be read afterwards with [ccusage](https://github.com/ryoppippi/ccusage). In a session of about 1.8 active hours, Claude Code used about 4.3 million input tokens per hour, over 90 % of them read from the cache, and about 44,000 output tokens per hour. With 5–10 hours of agent mode per week from the [survey](7-requirements.md#survey), i.e. 22–43 hours per month, Qwen3.6-35B-A3B on OpenRouter costs:

| Provider on OpenRouter | Per active hour | Per person and month |
|---|---|---|
| cheapest (Darkbloom) | about 0.14 $ | about 3–6 $ |
| typical (e.g. DeepInfra) | about 0.47 $ | about 10–20 $ |
| most expensive (AtlasCloud) | about 1.13 $ | about 25–49 $ |

*(Calculation with the prices of the providers on [OpenRouter](https://openrouter.ai/qwen/qwen3.6-35b-a3b), e.g. DeepInfra: 0.10 $ per million input and cached tokens, 0.95 $ per million output tokens)*

With a typical provider, this is about 80–160 $ per month for the 8 team members, compared to about 196 € for the subscription. The estimate is uncertain:

- The tokens per hour come from one session in which documentation was written. When programming, an agent reads more files and test output, so the usage can be higher.
- With training excluded, maybe not all providers can be used.
- Another model can need more or fewer tokens for the same task.

**Gemini CLI** is no longer compared: for free accounts, it was replaced by Antigravity CLI on 18 June 2026 *(Vendor: [Gemini CLI](https://geminicli.com/docs/resources/quota-and-pricing/))*.

Sources, as of 4 November 2026:

- Claude: [usage limits with Claude Code](https://support.claude.com/en/articles/11145838-using-claude-code-with-your-pro-or-max-plan), [model training](https://privacy.claude.com/en/articles/10023580-is-my-data-used-for-model-training), costs see [Requirements](7-requirements.md#costs-and-budget)
- GWDG Chat AI: [Chat AI](https://docs.hpc.gwdg.de/services/ai-services/chat-ai/index.html), [models](https://docs.hpc.gwdg.de/services/ai-services/chat-ai/models/index.html), [API and limits](https://docs.hpc.gwdg.de/services/ai-services/saia/index.html), [KISSKI LLM service](https://kisski.gwdg.de/en/leistungen/2-02-llm-service/), [data privacy](https://docs.hpc.gwdg.de/services/ai-services/chat-ai/faq/index.html)
- Usage-based API: [OpenRouter, Qwen3.6-35B-A3B](https://openrouter.ai/qwen/qwen3.6-35b-a3b), [OpenRouter, zero data retention](https://openrouter.ai/blog/insights/zero-data-retention/)
- GitHub Copilot: [plans](https://docs.github.com/en/copilot/concepts/billing/individual-plans), [students](https://docs.github.com/en/copilot/how-tos/copilot-on-github/set-up-copilot/enable-copilot/set-up-for-students), [changes for students](https://github.com/orgs/community/discussions/189268), training by default: [heise](https://www.heise.de/en/news/Only-with-opt-out-GitHub-will-train-Copilot-models-with-user-data-in-the-future-11226787.html?view=print), [The Register](https://www.theregister.com/2026/03/26/github_ai_training_policy_changes/)

### Must-haves

| Must-have | HTWG server | GWDG Chat AI | Usage-based API | GitHub Copilot Student |
|---|---|---|---|---|
| Agent mode | ✓ | ✓ | ✓ | ✓ |
| Context of at least 64K tokens | ✓ | ✓ | ✓ | depends on the model |
| Usable with VS Code or in the terminal | ✓ | ✓ | ✓ | ✓ |
| Cheaper than the current subscription | ✓ | ✓ | probably, about 10–20 $ per person and month | ✓ |
| Enough usage for 5–10 hours of agent mode per week | ✓ under our assumptions, see [Our resources](8-our-resources.md#reference-model-on-the-server) | unclear: an agent sends one request per step, the limit is 400 per day | ✓, limited only by costs | – 200 AI credits per month |
| Our code is not used for training | ✓ | ✓ | ✓ with settings | ✓ only with opt-out |

## Limits of this evaluation

- Only local operation on one laptop was tested briefly (see [Practical test](8-our-resources.md#practical-test)). The server, GWDG Chat AI, the usage-based API and GitHub Copilot Student were not tested; for them, the evaluation relies on benchmarks reported by the developers, reports of users and calculations.
- The tool around the model influences the result as much as the model itself (see [step 2](#2-a-capable-model)).
