# Engines

This page describes the engines we can use to run a model ourselves. It is only relevant for the operating models [Local](3-operating-models.md#local) and [Own server](3-operating-models.md#own-server).

## What an engine does

An engine loads a model into memory, runs it on the GPU or CPU and offers it through an API. The tool connects to this API with a base URL, e.g. `http://localhost:11434/v1` for Ollama. Most engines offer the OpenAI-compatible format, so almost every tool can connect to them.

With a subscription or a pay-per-token API, the provider runs the engine and we do not need one.

## Overview

| Engine | Kind | Operating systems | Licence |
|---|---|---|---|
| [Ollama](https://ollama.com) | Background service with command line | macOS, Windows, Linux | MIT |
| [LM Studio](https://lmstudio.ai) | Desktop app with user interface | macOS, Windows, Linux | Free, not open source |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | Command line and server (`llama-server`) | macOS, Windows, Linux | MIT |
| [MLX](https://github.com/ml-explore/mlx) | Library with server ([`mlx-lm`](https://github.com/ml-explore/mlx-lm)) | macOS (Apple Silicon); Linux (experimental) | MIT |
| [Strata](https://github.com/niko1221/strata) | Desktop app and command line | macOS (Apple Silicon), Windows, Linux | MIT |
| [vLLM](https://docs.vllm.ai) | Server for GPUs | Linux | Apache 2.0 |
| [SGLang](https://docs.sglang.io) | Server for GPUs | Linux | Apache 2.0 |

## Engines and hardware

An engine only uses the GPU if it supports the chips of its vendor. Without GPU support, the model runs on the CPU, which works but is slow.

| Engine | Mac (Apple Silicon) | NVIDIA GPU | AMD GPU | Intel GPU | CPU only |
|---|---|---|---|---|---|
| Ollama | ✓ | ✓ | ✓ | ✓ (Vulkan) | ✓ |
| LM Studio | ✓ | ✓ | ✓ | ✓ | ✓ |
| llama.cpp | ✓ | ✓ | ✓ | ✓ | ✓ |
| MLX | ✓ | ✓ (Linux, experimental) | – | – | ✓ (Linux) |
| Strata | ✓ | ✓ (RTX 20 and newer) | ✓ (RX 7000 and newer) | ✓ (Arc, Linux only) | – |
| vLLM | experimental | ✓ | ✓ | ✓ | ✓ |
| SGLang | – | ✓ | ✓ | – | ✓ (Intel Xeon) |

- **Desktop engines** (Ollama, LM Studio, llama.cpp, MLX, Strata) run on laptops and PCs. Engines built for one vendor's chips are faster there, e.g. MLX on Apple Silicon.
- **Server engines** (vLLM, SGLang) are built for Linux servers with GPUs and serve several users at the same time. A server engine must not be reachable from the internet without protection.
- The model and its context have to fit into memory, see [Hardware](6-hardware.md).

Sources: [Ollama](https://docs.ollama.com/gpu), [LM Studio](https://lmstudio.ai/docs/app/system-requirements), [llama.cpp](https://github.com/ggml-org/llama.cpp), [MLX](https://ml-explore.github.io/mlx/build/html/install.html), [Strata](https://github.com/niko1221/strata), [vLLM](https://docs.vllm.ai/en/latest/getting_started/installation/), [SGLang](https://docs.sglang.io).

## Engines and models

A model only runs on an engine that supports its architecture and its format. New models often bring a new architecture, and the engine needs an update first. Server engines are usually first.

| Engine | Model format | Where to find models |
|---|---|---|
| Ollama | GGUF | [Ollama library](https://ollama.com/library) |
| LM Studio | GGUF; MLX on Macs | [LM Studio models](https://lmstudio.ai/models), Hugging Face |
| llama.cpp | GGUF | [Hugging Face, GGUF](https://huggingface.co/models?library=gguf&sort=trending) |
| MLX | MLX | [mlx-community](https://huggingface.co/mlx-community) on Hugging Face |
| Strata | GGUF | A few selected models, see [Strata](https://github.com/niko1221/strata) |
| vLLM | Original weights from Hugging Face; quantized formats such as AWQ, GPTQ, FP8 | [Supported models](https://docs.vllm.ai/en/latest/models/supported_models.html) |
| SGLang | Like vLLM | [SGLang docs](https://docs.sglang.io) |

Popular models and current lists of them are on [Models](5-models.md#popular-models-october-2026).

## Engines and tools

Almost all engines offer an OpenAI-compatible API. That is why tools such as OpenCode, Cline, Roo Code, Aider or Codex CLI can connect to any of them. The tool only needs the base URL of the engine and the name of the model.

Exceptions:

- **Claude Code** needs an Anthropic-compatible API. Ollama, LM Studio and Strata offer one ([Ollama](https://docs.ollama.com/api/anthropic-compatibility), [LM Studio](https://lmstudio.ai/docs/developer/anthropic-compat)).
- **GitHub Copilot** can use local models through Ollama in chat and agent mode, but not for autocomplete ([VS Code docs](https://code.visualstudio.com/docs/copilot/customization/language-models)).

Two settings in the engine decide whether a tool works well in agent mode:

- **Context length:** Engines often start with a small default context. Agents need a large one, e.g. at least 64K tokens for Codex CLI according to the [Ollama docs](https://docs.ollama.com/integrations/codex).
- **Tool calling:** The engine has to pass tool calls between the tool and the model. This depends on the engine and on the model.
