# Engines

This page describes the engines we can use to run a model ourselves. For an overview of all building blocks, see [Overview](1-overview.md).

## What an engine does

An engine loads a model into memory, runs it on the GPU or CPU and offers it through an API. The tool connects to this API with a base URL, e.g. `http://localhost:11434/v1` for Ollama. Most engines offer the OpenAI-compatible format, so almost every tool can connect to them.

With a subscription or a pay-per-token API, the provider runs the engine and we do not need one.

## Overview

| Engine | Kind | Platforms | Model format | API | Licence |
|---|---|---|---|---|---|
| [Ollama](https://ollama.com) | Background service with command line | macOS, Windows, Linux | GGUF, from its own model library | OpenAI-compatible | MIT |
| [LM Studio](https://lmstudio.ai) | Desktop app with user interface | macOS, Windows, Linux | GGUF; MLX on Macs | OpenAI-compatible | Free, not open source |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | Command line and server (`llama-server`) | macOS, Windows, Linux | GGUF | OpenAI-compatible | MIT |
| [MLX](https://github.com/ml-explore/mlx) | Library with server (`mlx-lm`) | Apple Silicon only | MLX | OpenAI-compatible | MIT |
| [Strata](https://github.com/niko1221/strata) | Desktop app and command line | macOS (Apple Silicon), Windows, Linux | GGUF | OpenAI- and Anthropic-compatible | MIT |
| [vLLM](https://docs.vllm.ai) | Server for GPUs | Linux with GPU | Hugging Face formats | OpenAI-compatible | Apache 2.0 |
| [SGLang](https://docs.sglang.ai) | Server for GPUs | Linux with GPU | Hugging Face formats | OpenAI-compatible | Apache 2.0 |

## Which engine for which case

- **First steps on a laptop:** LM Studio for a user interface, Ollama for the command line. Both download models with one click or command.
- **Macs with Apple Silicon:** engines with MLX support (MLX itself or LM Studio) are often faster than GGUF on the same machine.
- **Claude Code with a local model:** needs an Anthropic-compatible API, e.g. Strata. It sets up coding tools with `strata connect`.
- **Server for the whole team:** vLLM or SGLang, because they serve several requests at the same time.

llama.cpp is the base for many other engines. It is only worth using directly if we need its settings in detail.

## What to watch out for

- **Context length:** Engines often start with a small default context. Agents need a large context, e.g. at least 64K tokens for Codex CLI according to the [Ollama docs](https://docs.ollama.com/integrations/codex). It has to be set in the engine.
- **Tool calling:** For agent mode, the engine has to pass tool calls between the tool and the model. This depends on the engine and on the model.
- **Memory:** The engine needs memory for the model and the context, see [Hardware](6-hardware.md).
- **Access:** An engine on a server must not be reachable from the internet without protection.
