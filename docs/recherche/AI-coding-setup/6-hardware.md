# Hardware

This page explains which hardware properties matter when we run a model ourselves. For an overview of all building blocks, see [Overview](1-overview.md).

!!! note "Hardware and engine belong together"
    Engines are often optimized for the chips of one vendor, e.g. MLX for Apple Silicon. Which engine runs on which hardware is described under [Engines and hardware](4-engines.md#engines-and-hardware).

## Memory

A model has to be loaded completely into memory before it can answer. Where it is loaded decides how fast it runs.

| Memory | What it is | Meaning for models |
|---|---|---|
| **RAM** | Main memory of the computer, used by the CPU | A model can run on the CPU from RAM without a graphics card. This works, but is slow. |
| **VRAM** | Memory on the graphics card, used by the GPU | Fast. The model should fit completely into VRAM; typical cards have 8–24 GB. |
| **Unified memory** | On Macs with Apple Silicon, CPU and GPU share one memory | A large part of the RAM can be used by the GPU. By default macOS does not give the whole memory to the GPU, so some of it stays reserved for the system. |

The operating system and other programs such as the IDE, the browser and Docker need memory at the same time.

How much memory a model needs, including its context, and how a model can be split between VRAM and RAM is described on Models: [Parameters and memory](5-models.md#parameters-and-memory), [Context](5-models.md#context) and [Offloading](5-models.md#offloading).

## Speed

Two values decide how fast a model feels:

- **Waiting time until the answer starts:** The model first processes the whole prompt, including the code the tool sends along. This mainly depends on the computing power of the GPU. With long contexts, as used by agents, this can take a noticeable time.
- **Tokens per second while answering:** For every generated token, the model reads its weights from memory. This mainly depends on the **memory bandwidth**. Rule of thumb: tokens per second ≈ memory bandwidth ÷ size of the model in memory.

For models with experts, only the active parameters count for the speed, see [Active parameters](5-models.md#active-parameters).
