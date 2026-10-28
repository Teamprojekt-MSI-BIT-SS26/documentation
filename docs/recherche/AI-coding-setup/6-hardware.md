# Hardware

This page explains which hardware properties matter when we run a model ourselves. For an overview of all building blocks, see [Overview](1-overview.md).

## Memory

A model has to be loaded completely into memory before it can answer. Where it is loaded decides how fast it runs.

| Memory | What it is | Meaning for models |
|---|---|---|
| **RAM** | Main memory of the computer, used by the CPU | A model can run on the CPU from RAM without a graphics card. This works, but is slow. |
| **VRAM** | Memory on the graphics card, used by the GPU | Fast. The model should fit completely into VRAM; typical cards have 8–24 GB. |
| **Unified memory** | On Macs with Apple Silicon, CPU and GPU share one memory | A large part of the RAM can be used by the GPU. By default macOS does not give the whole memory to the GPU, so some of it stays reserved for the system. |

If a model does not fit completely into VRAM, engines such as llama.cpp can split it: one part runs on the GPU, the rest on the CPU from RAM. This works, but the slow part slows down the whole model.

Besides the model itself, memory is needed for:

- **Context:** The longer the context, the more memory it uses (KV cache). Agents with large contexts need several additional GB.
- **Operating system and other programs:** IDE, browser and Docker keep running at the same time.

How much memory a model needs depends on its size and quantization, see [Quantization](5-models.md#quantization).

## Speed

Two values decide how fast a model feels:

- **Waiting time until the answer starts:** The model first processes the whole prompt, including the code the tool sends along. This mainly depends on the computing power of the GPU. With long contexts, as used by agents, this can take a noticeable time.
- **Tokens per second while answering:** For every generated token, the model reads its weights from memory. This mainly depends on the **memory bandwidth**. Rule of thumb: tokens per second ≈ memory bandwidth ÷ size of the model in memory.

Models with a mixture of experts (e.g. "A3B" in the name, see [Naming of model variants](5-models.md#naming-of-model-variants)) only read the active parameters per token. They need memory for the whole model, but answer as fast as a much smaller model.

## What this means for us

- Laptops without a strong graphics card or with little RAM can run small models for autocomplete or chat.
- Agent mode needs larger models and large contexts, and with them a lot of memory and a high memory bandwidth.
- Which models run at a usable speed on our laptops has to be tested with the hardware from the [survey](7-requirements.md#survey) (question 6).
