# Our resources

This page describes the resources available to our team, our laptops and a possible server, and what they allow. It applies the general information on [Models](5-models.md) and [Hardware](6-hardware.md); the laptops were collected in the [survey](7-requirements.md#results).

## Our devices

| Device | Count | Memory usable for models | Memory bandwidth |
|---|---|---|---|
| MacBook M1, 16 GB | 3 | about 12 GB | about 68 GB/s |
| MacBook M1 Pro, 16 GB | 1 | about 12 GB | 200 GB/s |
| Windows laptop, Intel i7 with integrated Intel Iris Xe Graphics, 16 GB | 1 | RAM, shared with the integrated graphics | about 50–90 GB/s |
| Windows laptop, AMD Ryzen 5000 with integrated AMD Radeon Graphics, 8 GB | 1 | RAM, shared with the integrated graphics | about 25–50 GB/s |

- **Macs:** By default, macOS lets the GPU use about 75 % of the unified memory. This value is measured by the community; Apple does not document it ([llmconfigurator](https://llmconfigurator.com/en/guides/troubleshooting/increase-metal-vram-limit-apple-silicon)).
- **Windows laptops:** Neither has a dedicated graphics card. The Intel Iris Xe Graphics and the AMD Radeon Graphics are built into the processor and share the RAM. Engines can use them through Vulkan, which speeds up processing long prompts, but the answer speed stays limited by the RAM bandwidth. The bandwidth depends on the memory type and is estimated.
- The IDE, browser and Docker use the same memory, so in practice less is available for the model and its context.

## What fits

Total memory = model + context + operating system and other programs (see [Parameters and memory](5-models.md#parameters-and-memory)). The sizes are those of the Q4_K_M versions in the Ollama library ([Qwen3.5](https://ollama.com/library/qwen3.5/tags), [Gemma 4](https://ollama.com/library/gemma4/tags)).

| Model (Q4_K_M) | Size | Mac 16 GB | Windows 16 GB | Windows 8 GB |
|---|---|---|---|---|
| Qwen3.5 4B | 3.3 GB | ✓ | ✓ | tight |
| Gemma 4 E4B | 6.6 GB | ✓ | ✓ | – |
| Qwen3.5 9B | 6.6 GB | ✓, but very slow in our test | ✓ | – |
| Gemma 4 12B | 8.0 GB | tight | tight | – |
| gpt-oss 20B, Qwen3.5 27B, Qwen3-Coder 30B-A3B, Gemma 4 26B | 13–22 GB | – | – | – |

With a model of about 7 GB, there is room for a context of a few GB, enough for questions and explanations, but not for an agent with a large context (see [Context](5-models.md#context)). In our [practical test](9-evaluation.md#practical-test) on a Mac with 32K context, macOS used swap even with the small models.

## Expected speed

Upper limits with the rule of thumb from [Hardware](6-hardware.md#example), for a model of 6.6 GB (e.g. Qwen3.5 9B):

| Device | Tokens per second (upper limit) |
|---|---|
| MacBook M1 | about 10 |
| MacBook M1 Pro | about 30 |
| Windows laptop, 16 GB | about 7–13 |

Real values are lower. Long prompts, as sent by agents, also take noticeably longer to process on these devices.

## Possible server

We might get access to a server with **2 × NVIDIA A100 40 GB**, together **80 GB VRAM**. Each card keeps its own memory, so the engine has to split a model across both cards:

- **Tensor parallelism** (vLLM, SGLang): each layer is split across both cards, which compute at the same time. Suited for several users ([vLLM docs](https://docs.vllm.ai/en/v0.8.4/serving/distributed_serving.html)).
- **Split by layers** (llama.cpp by default, Ollama): each card holds a part of the layers, and they compute one after the other. Suited for a single user ([llama.cpp server](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md), option `--split-mode`).

Each card has a memory bandwidth of about 1,500 GB/s, more than 20 times that of a MacBook M1.

What fits into 80 GB, with sizes from the Ollama library ([Qwen3-Coder](https://ollama.com/library/qwen3-coder/tags), [Qwen3-Coder-Next](https://ollama.com/library/qwen3-coder-next/tags), [gpt-oss](https://ollama.com/library/gpt-oss/tags), [Qwen3.5](https://ollama.com/library/qwen3.5/tags)):

| Model | Size | Fits into 80 GB |
|---|---|---|
| Qwen3-Coder 30B-A3B, Q4_K_M | 19 GB | ✓, with a large context for several users |
| Qwen3-Coder 30B-A3B, Q8_0 | 32 GB | ✓, with a large context |
| Qwen3-Coder-Next, Q4_K_M | 52 GB | ✓, with room for context |
| Qwen3-Coder 30B-A3B, 16 bits | 61 GB | tight |
| gpt-oss 120B | 65 GB | tight, little room for context |
| Qwen3.5 122B-A10B, Q4_K_M | 81 GB | – |
| Qwen3-Coder-Next, Q8_0 | 85 GB | – |

Coding models with agent mode and a large context fit, which is not possible on our laptops. The server would make agent mode possible with an open-weight model, without token costs and with our code staying at HTWG.

Still to be clarified with the operator of the server:

- whether we may run our own engine such as vLLM on it
- how we access it from outside, e.g. through VPN
- whether we share it with others

## Recommendation

- **Agent mode:** through a cloud option or the [possible server](#possible-server). Models that work reliably in agent mode, with a large context, do not fit on any of our laptops. This confirms decision 4 in the [requirements](7-requirements.md#decisions).
- **Local as an optional addition on the Macs:** a small model such as Qwen3.5 4B or Gemma 4 E4B for questions and explanations, with a limited context. Qwen3.5 9B was too slow in our [practical test](9-evaluation.md#practical-test). MLX versions are usually faster on Macs (see [Engines and models](4-engines.md#engines-and-models)).
- **Windows laptop with 16 GB:** at most small models for simple questions; slow.
- **Windows laptop with 8 GB:** local models are not recommended.
