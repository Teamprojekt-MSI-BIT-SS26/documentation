# Models

This page describes the models we can use, their variants and how much memory they need. For an overview of all building blocks, see [Overview](1-overview.md).

## Naming of model variants

Model names often contain hints about the variant:

| Part of the name | Meaning |
|---|---|
| **Coder** | Trained especially for programming |
| **Flash**, **Mini**, **Small** | Smaller variant: faster and cheaper, but weaker |
| **Instruct** | Trained to follow instructions in a chat |
| **30B**, **80B** | Total number of parameters in billions; a rough measure of size and memory needs |
| **A3B** | Only 3 billion parameters are active per step (mixture of experts); this makes the model faster, but it still needs memory for all parameters |

## Quantization

A model consists of billions of numbers (parameters). Normally each one is stored with 16 bits. Quantization stores them with fewer bits. The model gets smaller and faster, but loses some quality. Quantization only matters if we run the model ourselves.

| Bits | Memory for a model with 24 billion parameters | Quality |
|---|---|---|
| 16 | about 48 GB | original |
| 8 | about 24 GB | almost no loss |
| 4 | about 12 GB | small loss; the usual choice for local use |
| 3 | about 9 GB | noticeable loss |
| 2 | about 6 GB | large loss; only for experiments |

Rule of thumb: memory ≈ parameters × bits ÷ 8, plus a few GB for the context and the operating system. Quantized models are distributed in formats such as [GGUF](https://huggingface.co/docs/hub/gguf) (llama.cpp, Ollama, LM Studio) or MLX (Apple Silicon).
