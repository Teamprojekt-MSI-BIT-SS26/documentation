# Our resources

This page describes the resources available to our team, our laptops and a possible server at HTWG, and checks whether they can run a model for agent mode. It applies the general information on [Models](5-models.md) and [Hardware](6-hardware.md); the laptops were collected in the [survey](7-requirements.md#survey). The [Evaluation](9-evaluation.md) builds on the results.

## Reference model

As the reference for agent mode, we use **Qwen3.6-35B-A3B** (selected in [Evaluation, step 2](9-evaluation.md#2-a-capable-model)). According to its [model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B), it has 35B parameters, 3B of them active per token, reaches 73.4 % on SWE-bench Verified, supports a context of 262K tokens and is licensed under Apache 2.0.

| Part | Memory | Source |
|---|---|---|
| Model, Q4_K_M (GGUF, for Ollama and llama.cpp) | 23 GB | [Ollama](https://ollama.com/library/qwen3.6/tags) |
| Model, FP8 (for vLLM) | 37.5 GB | [Hugging Face](https://huggingface.co/Qwen/Qwen3.6-35B-A3B-FP8) |
| Model, original weights in BF16 | 72 GB | [Ollama](https://ollama.com/library/qwen3.6/tags) |
| Context of 64K tokens, 16 bits, per user | about 1.3 GB | Calculation, see below |

The context needs little memory, because only 10 of the 40 layers of this model use full attention ([model card](https://huggingface.co/Qwen/Qwen3.6-35B-A3B)): 2 × 10 layers × 2 KV heads × 256 × 2 bytes ≈ 20 KB per token, so 64K tokens need about 1.3 GB. *(Calculation from the [model configuration](https://huggingface.co/Qwen/Qwen3.6-35B-A3B/blob/main/config.json), method see [Context](5-models.md#memory-for-the-context))*

**Needed for agent mode: about 25 GB with Q4_K_M and 64K context, plus the operating system and other programs.**

## Our laptops

| Device | Count | Memory usable for models | Memory bandwidth |
|---|---|---|---|
| MacBook M1, 16 GB | 3 | about 12 GB | about 68 GB/s |
| MacBook M1 Pro, 16 GB | 1 | about 12 GB | 200 GB/s |
| Windows laptop, Intel i7 with integrated Intel Iris Xe Graphics, 16 GB | 1 | RAM, shared with the integrated graphics | about 50–90 GB/s |
| Windows laptop, AMD Ryzen 5000 with integrated AMD Radeon Graphics, 8 GB | 1 | RAM, shared with the integrated graphics | about 25–50 GB/s |

- **Macs:** By default, macOS lets the GPU use about 75 % of the unified memory. This value is measured by the community; Apple does not document it ([llmconfigurator](https://llmconfigurator.com/en/guides/troubleshooting/increase-metal-vram-limit-apple-silicon)).
- **Windows laptops:** Neither has a dedicated graphics card. The Intel Iris Xe Graphics and the AMD Radeon Graphics are built into the processor and share the RAM. Engines can use them through Vulkan, which speeds up processing long prompts, but the answer speed stays limited by the RAM bandwidth. The bandwidth depends on the memory type and is estimated.
- The IDE, browser and Docker use the same memory, so in practice less is available for the model and its context.

### What fits

Total memory = model + context + operating system and other programs (see [Parameters and memory](5-models.md#parameters-and-memory)). The sizes are those of the Q4_K_M versions in the Ollama library ([Qwen3.5](https://ollama.com/library/qwen3.5/tags), [Gemma 4](https://ollama.com/library/gemma4/tags), [Qwen3.6](https://ollama.com/library/qwen3.6/tags)).

| Model (Q4_K_M) | Size | Mac 16 GB | Windows 16 GB | Windows 8 GB |
|---|---|---|---|---|
| Qwen3.5 4B | 3.3 GB | ✓ | ✓ | tight |
| Gemma 4 E4B | 6.6 GB | ✓ | ✓ | – |
| Qwen3.5 9B | 6.6 GB | ✓ | ✓ | – |
| Gemma 4 12B | 8.0 GB | tight | tight | – |
| Qwen3.5 27B, Gemma 4 26B | 17–18 GB | – | – | – |
| **Qwen3.6-35B-A3B (reference model)** | 23 GB | – | – | – |

- **The reference model does not fit on any laptop.** It needs about 25 GB, the laptops have at most 16 GB in total. [Offloading](5-models.md#offloading) does not help, because RAM and VRAM together still have to hold the whole model.
- **Even the smallest suitable model does not fit.** The smallest model above the threshold from [Evaluation, step 2](9-evaluation.md#2-a-capable-model) is Qwen3.6-27B (77.2 % on SWE-bench Verified, [model card](https://huggingface.co/Qwen/Qwen3.6-27B)). With Q4_K_M it needs 17 GB ([Ollama](https://ollama.com/library/qwen3.6/tags)) plus context, more than the laptops have. Devstral Small 2 (24B, 15 GB), slightly below the threshold, does not fit into the about 12 GB the GPU can use on our Macs either ([Ollama](https://ollama.com/library/devstral-small-2/tags)). The result therefore does not depend on the choice of the reference model.
- **The models that fit are not proven to be good enough.** For Qwen3.5 4B and 9B and Gemma 4 E4B and 12B, the developers report no SWE-bench Verified value ([Qwen3.5 9B](https://huggingface.co/Qwen/Qwen3.5-9B), [Gemma 4 12B](https://huggingface.co/google/gemma-4-12b-it)). The community recommends models of this size for autocomplete and chat, but not for agentic coding, mainly because their tool calling is unreliable *(Community: [InsiderLLM](https://insiderllm.com/guides/best-local-coding-models-2026/))*.
- **Little room for context.** With a model of about 7 GB, there is room for a context of a few GB, enough for questions and explanations, but not for an agent with a large context (see [Context](5-models.md#how-much-context-for-what)).

### Expected speed

Upper limits with the rule of thumb from [Hardware](6-hardware.md#example), for a model of 6.6 GB (e.g. Qwen3.5 9B):

| Device | Tokens per second (upper limit) |
|---|---|
| MacBook M1 | about 10 |
| MacBook M1 Pro | about 30 |
| Windows laptop, 16 GB | about 7–13 |

The Windows laptop with 8 GB is missing, because a model of this size does not fit. Real values are lower. Long prompts, as sent by agents, also take noticeably longer to process on these devices.

### Practical test

To check these estimates, we ran a short test on a laptop:

- **Device:** MacBook M1 Pro, 16 GB
- **Engine and tool:** LM Studio with OpenCode, context set to 32K tokens
- **Other programs:** all closed except Docker, which used 3.5 GB
- **Task:** give an overview of this documentation repository and write it into a Markdown file

| Model | Context | Time until the result | Quality | Swap used by the system |
|---|---|---|---|---|
| Qwen3.5 4B (MLX, 4 bits) | 32K | 2 minutes 30 seconds | good | yes |
| Qwen3.5 9B (4 bits) | 32K | 33 minutes | overengineered | yes |
| Gemma 4 E4B (4 bits) | 32K | 1 minute 44 seconds | sufficient | yes |
| Gemma 4 12B (4 bits) | 32K | 5 minutes 24 seconds | good | yes |

What the test shows:

- **Memory gets tight even with small models.** In all four tests, macOS used swap, i.e. moved parts of the memory to the SSD, with only Docker (3.5 GB) running besides the model. Whether the model itself caused the swap is unclear, but memory was tight. In daily work, the IDE and browser need memory as well.
- **The 9B model was very slow.** It took more than ten times as long as the 4B model. Possible reasons are swap, because the SSD is much slower than memory (see [Speed](6-hardware.md#speed)), and the full context described below.
- **32K context is not enough for an agent.** With the 9B model, the context ran full and the engine cut out its middle several times. The model lost parts of the task and the result became overengineered. This supports the requirement of at least 64K tokens and matches [Lost in the Middle](https://arxiv.org/abs/2307.03172).
- **Small tasks can work locally.** Qwen3.5 4B and Gemma 4 E4B delivered usable results within about two minutes, Gemma 4 12B a good result within about five minutes.

The test ran only on the M1 Pro, the fastest of our laptops. On the M1 and the Windows laptops, it would be slower.

### Result for the laptops

- **Agent mode: not possible.** Neither the reference model nor the smallest suitable model fits; for the models that fit, there is no evidence that they are good enough, and their context is too small.
- **Small tasks: possible on the Macs.** Small models such as Qwen3.5 4B, Gemma 4 E4B or Gemma 4 12B answer simple questions with a limited context. MLX versions are usually faster on Macs (see [Engines and models](4-engines.md#engines-and-models)).
- **Windows laptop with 16 GB:** at most small models for simple questions; slow.
- **Windows laptop with 8 GB:** local models are not recommended.

## Possible server

The Institute for Optical Systems (IOS) of HTWG runs a GPU cluster with **8 × NVIDIA A100 40 GB**, together 320 GB VRAM, and 192 CPU cores. According to the article, it is initially used by professors, doctoral candidates and student assistants; later, students may also use it for theses *(HTWG: [Gamer würden neidisch werden](https://www.htwg-konstanz.de/hochschule/magazin/gamer-wuerden-neidisch-werden), 18 February 2021; [IOS](https://www.ios.htwg-konstanz.de/))*.

What exactly is installed today and what we could use is not confirmed. **We assume that we can use 2 of the A100 40 GB cards, together 80 GB VRAM.**

### What the hardware allows

- **Bandwidth:** each card has a memory bandwidth of about 1,500 GB/s ([Wikipedia](https://en.wikipedia.org/wiki/Ampere_(microarchitecture))), more than 20 times that of a MacBook M1.
- **Splitting across two cards:** each card keeps its own memory, so the engine has to split the model. vLLM uses tensor parallelism: each layer is split across both cards, which compute at the same time. This is the recommended way if a model does not fit on one GPU but on one server with several GPUs ([vLLM](https://docs.vllm.ai/en/latest/serving/parallelism_scaling/)).
- **Format:** vLLM runs the original weights (BF16) or an FP8 version. The A100 cannot compute in FP8, but vLLM supports FP8 models on it as weight-only FP8 with Marlin kernels ([vLLM](https://docs.vllm.ai/en/latest/features/quantization/llm_compressor/fp8/)). The GGUF versions from the Ollama library are meant for Ollama and llama.cpp.

### Reference model on the server

| Setup | Memory | Fits into 80 GB |
|---|---|---|
| FP8 + 64K context for 1 user | about 39 GB | ✓, about 41 GB free |
| FP8 + 64K context for 8 users | about 48 GB | ✓, about 32 GB free |
| BF16 + 64K context for 1 user | about 73 GB | tight, about 7 GB free |
| BF16 + 64K context for 8 users | about 83 GB | – |

*(Calculation from the [reference model](#reference-model))* The engine itself also needs some memory, so the free memory is smaller in practice.

**Speed:** per token, only the 3B active parameters are read, about 3 GB in FP8. With the rule of thumb from [Hardware](6-hardware.md#speed), the upper limit is several hundred tokens per second (1,500 GB/s ÷ 3 GB ≈ 500), compared to about 10 on a MacBook M1 with a much smaller model.

### Still to be clarified with IOS

- which cards are installed today and how many we may use
- whether we may run our own engine such as vLLM on it
- how we access it from outside, e.g. through VPN
- whether we share it with others, and how much capacity is left for us

### Result for the server

Under our assumption of 2 × A100 40 GB, the server can run the reference model in FP8 with a context of 64K tokens for the whole team at the same time, with memory to spare. It would therefore be a workable option for agent mode, without token costs and with our code staying at HTWG. Whether we get access is still open.

## Summary

| Resource | Agent mode | Small tasks |
|---|---|---|
| MacBooks, 16 GB | – | ✓ with small models |
| Windows laptop, 16 GB | – | at most, slow |
| Windows laptop, 8 GB | – | – |
| HTWG server, 2 × A100 40 GB (assumed) | ✓ with the reference model in FP8 | ✓ |
