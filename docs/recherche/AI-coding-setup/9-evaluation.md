# Evaluation

## Result

**Recommendation:** an open-weight model, Qwen3.6-35B-A3B, with OpenCode or Cline instead of the Claude subscription. The model can be used in three ways; the tool stays the same, only the address of the API changes.

| # | Option | Costs | Assessment |
|---|---|---|---|
| 1 | **HTWG server** (IOS) | none | Best option, if we get access: no limits, our code stays at HTWG. Access is not clarified yet. |
| 2 | **GWDG Chat AI** (Academic Cloud) | none | Same model, free, data is not stored. Tested: small tasks as good as Claude, a larger task only partly correct. Limited to 400 requests per day and 3,000 per month per API key; the three test tasks needed 48 requests. |
| 3 | **Usage-based API** (OpenRouter) | about 10–20 $ per person and month (estimate) | Same model without limits, training can be excluded. Probably cheaper than the subscription, but the estimate is still uncertain. |
| – | GitHub Copilot Student | none | Not recommended for agent mode: 200 AI credits per month, model chosen automatically. Not tested yet. |
| – | Gemini CLI | – | Dropped: the free tier was replaced by Antigravity CLI in June 2026. |

- **Quality:** Qwen3.6-35B-A3B is slightly below Claude Sonnet 4.5 from September 2025 ([step 4](#4-comparison-of-the-options)). Our [test](#5-test-task) confirms a noticeable drop compared to Claude Sonnet 5.5: on the small task both were equally good, on the larger task Qwen ignored part of the requirements and opened unprotected endpoints that were not asked for. Its results have to be reviewed carefully.
- **Laptops:** not suitable for agent mode. In our test, Gemma 4 12B on a MacBook M1 Pro needed 24 minutes for the small task and failed the other two. Local models are at most an addition for short questions in chat ([Our resources](8-our-resources.md#summary)).

**Next steps:**

1. Ask IOS whether and how we can use the server.
2. Test the usage-based API and GitHub Copilot Student with the same [test tasks](#5-test-task).
3. Read the token usage of team members who program with [ccusage](https://github.com/ryoppippi/ccusage) to confirm the cost estimate in [step 4](#4-comparison-of-the-options).
4. Let the team work with GWDG Chat AI for a while, to see whether the quality and the request limit are enough in daily work.

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

- **Laptops:** not suitable for agent mode. The reference model needs about 25 GB with Q4_K_M and 64K context, the laptops have at most 16 GB. On the Macs, small models are at most enough for short questions in chat ([Result for the laptops](8-our-resources.md#result-for-the-laptops)); the [test](#5-test-task) confirms this.
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

## 5. Test task

Every option solves the same three tasks from the Teamable code. They cover the main uses from the [survey](7-requirements.md#survey): explaining code, debugging and writing code. Claude Code solves them as well, as the reference.

| # | Task | Mode | Repository and commit |
|---|---|---|---|
| 1 | **Explain code:** the OAuth flow between Canvas and the backend, and why it fails for `fetch` calls from the frontend | chat, read only | `teamable-backend`, `183a36c` |
| 2 | **Fix a bug:** `authenticate/route.ts` returns a plain object instead of a `Response` when parameters are missing; return HTTP 400 and add a Jest test | agent, small | `teamable-frontend`, `98ce1c8` |
| 3 | **Implement a feature:** the missing endpoint `GET /api/v1/teamset-templates` for the page *Project sets*, with Django tests | agent, several files | `teamable-backend`, `183a36c` |

The expected results are known: task 1 is described in [Backend: LMS integration](../../current-state/backend-lms-integration.md), the bug and the missing endpoint of tasks 2 and 3 in [Frontend: LMS integration](../../current-state/frontend-lms-integration.md).

??? info "Prompts"
    The prompts are given word for word, without further help.

    1. Explain how the OAuth flow between Canvas and this backend works (see src/canvas_oauth/). Describe the steps from the first request to a stored token, and explain why the flow fails when the frontend calls the API with fetch. Do not change any files.
    2. In src/app/(accounts)/authenticate/route.ts, the GET handler returns a plain object instead of a Response when the query parameters token or path are missing. Fix it so that it returns an HTTP 400 response. Add a Jest test for this case in the test/ folder and make sure `npx jest` passes.
    3. The frontend page "Project sets" calls GET /api/v1/teamset-templates and expects a list of team set templates, each with id, name, teams (the team templates of the set), createdAt and updatedAt. This endpoint does not exist in the backend. Implement it using the existing TeamSetTemplate and TeamTemplate models, and add Django tests for it. The tests must pass with `cd src && ../venv/bin/python manage.py test`.

**Procedure:**

- Every run starts on the commit above, in its own branch or worktree, with a new session.
- The prompt is given word for word. If we have to step in, this is noted.
- If the agent asks for the frontend in task 3, the answer is: "The frontend code is not available in this repository. Use the fields described in the task: id, name, teams (the team templates of the set), createdAt and updatedAt."
- Only test data, e.g. from `makefakeclass`; no real data of students from Moodle or Canvas.

**Rating per task and option:**

| Criterion | How it is measured |
|---|---|
| Result | done / partly / not done; do the tests pass? |
| Code quality | 1–5, with a short reason |
| Interventions | how often we had to correct or ask again |
| Time | from the prompt to the finished result |
| Usage | tokens and costs with [ccusage](https://github.com/ryoppippi/ccusage); for GWDG the number of requests, because of the limit of 400 per day |

### Results

| Option | Task 1 | Task 2 | Task 3 | Setup effort |
|---|---|---|---|---|
| Claude Code (reference) | done; quality 5/5; no interventions; 35 s; 5 requests, 209K input tokens (90 % from the cache) and 3K output tokens | done, tests pass; quality 5/5; no interventions; 20 s; 4 requests, 154K input tokens (90 % from the cache) and 1K output tokens | done, tests pass; quality 5/5; no interventions; 71 s; 9 requests, 407K input tokens (93 % from the cache) and 4K output tokens | none, in use |
| GWDG Chat AI with OpenCode | done; quality 3/5; no interventions; 27 s; 6 requests, 80K input and 5K output tokens | done, tests pass; quality 4/5; no interventions; 19 s; 7 requests, 85K input and 3K output tokens | partly done, tests pass; quality 2/5; 1 question answered; 3 min 17 s; 35 requests, 807K input and 19K output tokens | low: API key as environment variable and `opencode.json`, see [Setup guide](10-setup-guide.md) |
| Usage-based API with OpenCode | not tested yet | not tested yet | not tested yet | |
| GitHub Copilot Student | not tested yet | not tested yet | not tested yet | |
| Local: Gemma 4 12B on MacBook M1 Pro, 32K context | not done: stopped after 25 min without an answer; 5 steps, single steps took up to 12 min | done, tests pass; quality 4/5; no interventions; 24 min; 35 requests, 371K input and 8K output tokens; context compacted twice | not done: stopped after 25 min without any change; only read files, context compacted once | low: LM Studio and `opencode.json` |
| HTWG server | not testable yet, no access | | | |

Notes on the runs (4 November 2026), Claude Code with Claude Sonnet 5.5:

- **Claude, task 1:** The explanation matches the code and our [documentation](../../current-state/backend-lms-integration.md): all steps from the missing token through the redirect to Canvas, the callback with the `state` check, to the stored token and the DRF token for the frontend. For `fetch`, it names the redirect to Canvas' login page that is blocked by CORS, the missing session cookie and the wrong final redirect, and proposes a fix. It also found a real bug: `oauth_callback` uses `create` on a one-to-one relation, so a second authorization fails. No files were changed.
- **Claude, task 2:** The fix returns a plain-text `Response` with status 400. The test checks the three cases as separate tests and also that no cookie is set and no redirect happens. It mocks the Next.js functions, so no change to the Jest configuration was needed.
- **Claude, task 3:** The endpoint returns exactly the requested fields, including `createdAt` and `updatedAt`, is read-only and loads the team templates in one query. It requires a login, like the other endpoints of the project, and also answers without a trailing slash, as the frontend calls it. The 5 tests cover the fields, the empty list, the login and the read-only access. It did not ask for the frontend. Note: the page *Project sets* currently calls the endpoint without a token, so it has to send one.
- **GWDG, task 1:** The steps of the flow are described correctly and in detail, including the token refresh. The explanation for `fetch` stays general ("fetch cannot show the Canvas page") and does not name CORS. It contains two errors: the frontend authenticates with the DRF token, not with a session cookie, and `initiate_oauth_check` is described as broken for the first login, although the flow returns to it after the callback. It proposes no fix and did not find the bug in `oauth_callback`.
- **GWDG, task 2:** The fix is correct and minimal (`Response.json(..., { status: 400 })`). To import the route in the test, the agent added the missing `@/` alias to the Jest configuration, which is sensible. Minor points: all three cases are in one test instead of three, and an unused import remains. GWDG reported no cached tokens, so every request sent the whole context again.
- **GWDG, task 3:** The endpoint works and returns the team templates; the agent wrote 6 tests that pass. But it returns `created_at` and `updated_at` instead of the requested `createdAt` and `updatedAt`, and it registered three more endpoints that were not asked for (`teamsets`, `team-templates`, `teams`). As the project sets no permission classes, these allow anyone to create, change and delete data. The agent asked once where the frontend code is; the time includes about one minute until it was answered. It started a subagent to explore the code; the requests of both are counted. No rate limit was hit.

- **Local, task 2:** The fix is the same as Claude's, and the three cases are separate tests. Minor points: one test name does not match its case, and the request is cast with `as any`. The run took 24 minutes instead of 20 seconds: OpenCode compacted the context twice, after the second compaction the agent ran many shell commands in a row, and one step took about 8 minutes. A broken control token (`<|channel>`) appeared once in the output. During the run, 15.25 of 16 GB were used, 11.7 GB of it by the model as wired memory, and macOS used 3.1 GB of swap. LM Studio had loaded the model with its default of 4 parallel slots.
- **Local, task 1:** Stopped after 25 minutes without an answer. In this time, the agent ran one shell command and read four files; the shell command alone took 12 minutes, one read 8.5 minutes. 14.0 of 16 GB were used, 10.7 GB of it by the model, and macOS used 4.0 GB of swap, so parts of the memory had to be read from the SSD (see [Speed](6-hardware.md#speed)). The swap was already in use from the earlier runs; the Mac was not restarted between the tests.
- **Local, task 3:** Stopped after 25 minutes without changing any file. The agent read nine files in six minutes, then the context of 32K was compacted, and the next step took ten minutes without a result. The model was loaded with one parallel slot; 15.5 of 16 GB were used, 13 GB of it by the model, with 2.7 GB of swap. Two earlier attempts are not counted: the first hung at the first step with 4 parallel slots, the second ended after one answer without a tool call while the first was still running in the background.

## Limits of this evaluation

- GWDG Chat AI, Claude Code and a local model were tested with three [test tasks](#5-test-task), each once and by one person. The server, the usage-based API and GitHub Copilot Student were not tested yet; for them, the evaluation relies on benchmarks reported by the developers, reports of users and calculations.
- The local tests ran one after the other without restarting the Mac, so swap from earlier runs was still in use.
- The tool around the model influences the result as much as the model itself (see [step 2](#2-a-capable-model)).
