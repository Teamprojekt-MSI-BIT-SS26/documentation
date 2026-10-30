# Evaluation

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

Open-weight models recommended by the community for local agentic coding *(Community: [InsiderLLM](https://insiderllm.com/guides/best-local-coding-models-2026/), [Latent Space](https://latent.space/p/ainews-top-local-models-list-april/))*:

| Model | Type | SWE-bench Verified | Source of the value |
|---|---|---|---|
| Qwen3.6-27B | dense, 27B | 77.2 % | [Model card](https://huggingface.co/Qwen/Qwen3.6-27B) |
| Qwen3.6-35B-A3B | mixture of experts, 3B active | 73.4 % | [Model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B) |
| Qwen3-Coder-Next | mixture of experts, 80B, 3B active | 70.6 % | [Technical report](https://arxiv.org/abs/2603.00729) |
| Devstral Small 2 | dense, 24B | 68.0 % | Mistral, reported by [llmrun.dev](https://llmrun.dev/benchmark/swe-bench-verified?show=all) |

Smaller models are hardly recommended for agentic coding, mainly because their tool calling is unreliable. We therefore require **about 65–70 % on SWE-bench Verified and native tool calling**.

**Reference model: Qwen3.6-35B-A3B.** It is clearly above the threshold, fast thanks to only 3B active parameters, has a context of 262K tokens and is licensed under Apache 2.0. *(Vendor: [model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B))*

!!! note "Limits of benchmarks"
    The values are reported by the developers. The result also depends on the tool around the model, so values of the same model can differ. *(Study: [Coding benchmarks are misaligned with agentic software engineering](https://arxiv.org/abs/2606.17799))*

## 3. Hardware, engine and model must fit together

The reference model only runs if an engine supports its architecture on the given hardware (see [Engines](4-engines.md#engines-and-hardware)). For Qwen3.6-35B-A3B:

| Hardware | Engine | Format | Source |
|---|---|---|---|
| Mac with Apple Silicon | MLX, LM Studio | MLX | [mlx-community](https://huggingface.co/mlx-community/Qwen3.6-35B-A3B-4bit) |
| Mac, Windows, Linux | Ollama, llama.cpp, LM Studio | GGUF | [Ollama](https://ollama.com/library/qwen3.6/tags), [unsloth](https://huggingface.co/unsloth/Qwen3.6-35B-A3B-GGUF) |
| Server with NVIDIA GPUs | vLLM | original weights | [vLLM supported models](https://docs.vllm.ai/en/latest/models/supported_models.html) |

The model is supported on all our platforms, so the engine is not the limiting factor. The limit is the memory.

## 4. Memory needed

| Part | Memory | Source |
|---|---|---|
| Model, Q4_K_M | 24 GB | Vendor: [Ollama](https://ollama.com/library/qwen3.6/tags) |
| Model, Q8_0 | 39 GB | Vendor: [Ollama](https://ollama.com/library/qwen3.6/tags) |
| Context of 64K tokens, 16 bits | about 1.3 GB | Calculation |
| Operating system and other programs | a few GB | |

**Total with Q4_K_M and 64K context: about 26 GB plus the operating system and other programs.**

The context needs little memory, because only 10 of the 40 layers of this model use full attention: 2 × 10 layers × 2 KV heads × 256 × 2 bytes ≈ 20 KB per token, so 64K tokens need about 1.3 GB. *(Calculation from the [model configuration](https://huggingface.co/Qwen/Qwen3.6-35B-A3B/blob/main/config.json), method see [Context](5-models.md#memory-for-the-context))*

With [offloading](5-models.md#offloading), the experts can be moved from VRAM to RAM. This lowers the need for VRAM, but VRAM and RAM together still have to hold the whole model.

## 5. Our laptops are not enough

| Device | Memory | Reference model with 64K context |
|---|---|---|
| MacBook M1 / M1 Pro, 16 GB | about 12 GB usable by the GPU, 16 GB in total | – does not fit, even with offloading |
| Windows laptop, 16 GB | 16 GB RAM, shared with the integrated graphics | – does not fit |
| Windows laptop, 8 GB | 8 GB RAM, shared with the integrated graphics | – does not fit |

All laptops have at most 16 GB in total, but the reference model needs about 26 GB. Models that fit are clearly smaller and below the threshold from step 2. Local operation is therefore ruled out for agent mode. *(Calculation, see [Our resources](8-our-resources.md))*

## 6. The possible server is enough

The [possible server](8-our-resources.md#possible-server) has 2 × A100 with 40 GB each, together 80 GB VRAM. vLLM splits the model across both cards.

| Setup | Memory | Fits into 80 GB |
|---|---|---|
| Reference model Q8_0 + 64K context for 1 user | about 40 GB | ✓ |
| Reference model Q8_0 + 64K context for 8 users | about 50 GB | ✓ |

*(Calculation from steps 4 and 5)*

## 7. Comparison of the remaining options

| | HTWG server | Pay per token | GitHub Copilot Student | Gemini CLI |
|---|---|---|---|---|
| **Model** | Qwen3.6-35B-A3B | Qwen3.6-35B-A3B | chosen automatically | Gemini models |
| **Costs** | none for us, if HTWG provides the server | 0.10 $ per million input tokens, 0.90 $ per million output tokens | free for verified students | free |
| **Limits** | capacity of the two GPUs | budget | allowance of AI credits | 1,000 requests per day |
| **Our code used for training** | no, it stays at HTWG | can be excluded (`data_collection: deny`, zero data retention) | yes by default since 24 April 2026, opt-out in the settings | possible with a personal account; setting "Usage Statistics" |
| **Open points** | access, own engine, shared use | token usage per person | | |

Sources, as of 30 October 2026:

- Pay per token: [OpenRouter, Qwen3.6-35B-A3B](https://openrouter.ai/qwen/qwen3.6-35b-a3b), [OpenRouter, zero data retention](https://openrouter.ai/blog/insights/zero-data-retention/)
- GitHub Copilot: [plans](https://docs.github.com/en/copilot/concepts/billing/individual-plans), [students](https://docs.github.com/en/copilot/how-tos/copilot-on-github/set-up-copilot/enable-copilot/set-up-for-students), training by default: [heise](https://www.heise.de/en/news/Only-with-opt-out-GitHub-will-train-Copilot-models-with-user-data-in-the-future-11226787.html?view=print), [The Register](https://www.theregister.com/2026/03/26/github_ai_training_policy_changes/)
- Gemini CLI: [quotas](https://geminicli.com/docs/resources/quota-and-pricing/), [terms and privacy](https://geminicli.com/docs/tos-privacy)

The costs for pay per token depend on how many tokens we use. To estimate them, the token usage of our current subscription is needed.

GitHub Copilot and Gemini CLI cannot use the reference model. They are compared through the must-haves.

### Must-haves

| Must-have | HTWG server | Pay per token | GitHub Copilot Student | Gemini CLI |
|---|---|---|---|---|
| Agent mode | ✓ | ✓ | ✓ | ✓ |
| Context of at least 64K tokens | ✓ | ✓ | depends on the model | ✓ |
| Usable with VS Code or in the terminal | ✓ | ✓ | ✓ | ✓ |
| Costs within the budget | ✓ | depends on usage | ✓ | ✓ |
| Our code is not used for training | ✓ | ✓ with settings | ✓ only with opt-out | unclear |

## Practical test

To check steps 1 and 5, we ran a short test on a laptop:

- **Device:** MacBook M1 Pro, 16 GB
- **Engine and tool:** LM Studio with OpenCode, context set to 32K tokens
- **Other programs:** all closed except Docker, which used 3.5 GB
- **Task:** give an overview of this documentation repository and write it into a Markdown file

| Model | Context | Time until the result | Quality | Swap used by the system |
|---|---|---|---|---|
| Qwen3.5 4B (MLX, 4 bits) | 32K | 2 minutes 30 seconds | good | yes |
| Qwen3.5 9B (4 bits) | 32K | 33 minutes | overengineered | yes |
| Gemma 4 E4B (4 bits) | 32K | 1 minute 44 seconds | sufficient | yes |

What the test shows:

- **Memory gets tight even with small models.** In all three tests, macOS used swap, i.e. moved parts of the memory to the SSD, with only Docker (3.5 GB) running besides the model. Whether the model itself caused the swap is unclear, but memory was tight. In daily work, the IDE and browser need memory as well. The reference model needs about 26 GB and is therefore clearly out of reach (step 5).
- **The 9B model was very slow.** It took more than ten times as long as the 4B model. Possible reasons are swap, because the SSD is much slower than memory (see [Memory bandwidth](6-hardware.md#memory-bandwidth)), and the full context described below.
- **32K context is not enough for an agent.** With the 9B model, the context ran full and the engine cut out its middle several times. The model lost parts of the task and the result became overengineered. This supports the requirement of at least 64K tokens (step 1) and matches *Lost in the Middle*.
- **Small tasks can work locally.** The small models (Qwen3.5 4B, Gemma 4 E4B) delivered usable results for this small task, which matches local models as an optional addition (see [Our resources](8-our-resources.md#recommendation)).

## Recommendation

1. **HTWG server with the reference model**, if we get access. It meets all must-haves, causes no costs for us, and our code stays at HTWG. Qwen3.6-35B-A3B runs with vLLM on both A100 cards, with enough memory for a large context for the whole team.
2. **Pay per token with the same model** until the server is available, or if it is not. Qwen3.6-35B-A3B through OpenRouter, with training excluded (`data_collection: deny`, zero data retention). Tools such as OpenCode or Cline can be used for both options; switching to the server later only changes the address of the API. The costs still have to be estimated from our token usage.
3. **GitHub Copilot Student only as a fallback.** It is free, but the model is chosen automatically, the usage is limited by AI credits, and training with our data has to be switched off by every team member.
4. **Gemini CLI is not recommended** for our code, because it is unclear whether our code can be fully excluded from training.

**Local models** are at most an optional addition on the Macs for simple questions, e.g. Gemma 4 E4B or Qwen3.5 4B (see [Practical test](#practical-test)).

Before the whole team switches, the chosen option should be tested briefly in practice.

## Limits of this evaluation

- Only local operation on one laptop was tested briefly (see [Practical test](#practical-test)). The server, pay per token and the subscriptions were not tested; for them, the evaluation relies on benchmarks reported by the developers and on calculations.
- The tool around the model influences the result as much as the model itself (see step 2).
- A short practical test of the chosen option is still recommended before the whole team switches.
