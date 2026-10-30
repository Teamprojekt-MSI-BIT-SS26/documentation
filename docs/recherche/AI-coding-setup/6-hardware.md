# Hardware

This page explains which hardware properties matter when we run a model ourselves.

!!! note "Hardware and engine belong together"
    Engines are often optimized for the chips of one vendor, e.g. MLX for Apple Silicon. Which engine runs on which hardware is described under [Engines and hardware](4-engines.md#engines-and-hardware).

## Memory

A model has to be loaded completely into memory before it can answer. Where it is loaded decides how fast it runs.

| Memory | What it is | Meaning for models |
|---|---|---|
| **RAM** | Main memory of the computer, used by the CPU | A model can run on the CPU from RAM without a graphics card. This works, but is slow. |
| **VRAM** | Memory on the graphics card, used by the GPU | Fast. The model should fit completely into VRAM; typical cards have 8–24 GB. |
| **Unified memory** | On Macs with Apple Silicon, CPU and GPU share one memory | A large part of the RAM can be used by the GPU. By default macOS does not give the whole memory to the GPU, so some of it stays reserved for the system. |
| **SSD** | Storage for files, including the model files | Slowest. Models are loaded from here into memory, and model files need a lot of free space. Some engines read parts of very large mixture-of-experts models directly from the SSD while answering, e.g. Strata for DeepSeek. This works, but is much slower than RAM (see [Offloading](5-models.md#offloading)). |

The operating system and other programs such as the IDE, the browser and Docker need memory at the same time.

How much memory a model needs, including its context, and how a model can be split between VRAM and RAM is described on Models: [Parameters and memory](5-models.md#parameters-and-memory), [Context](5-models.md#context) and [Offloading](5-models.md#offloading).

## Speed

Two values decide how fast a model feels:

- **Waiting time until the answer starts:** The model first processes the whole prompt, including the code the tool sends along. This mainly depends on the computing power of the GPU. With long contexts, as used by agents, this can take a noticeable time.
- **Tokens per second while answering:** For every generated token, the model reads its weights from memory. This mainly depends on the **memory bandwidth**. Rule of thumb: tokens per second ≈ memory bandwidth ÷ size of the model in memory.

### Memory bandwidth

Typical values:

| Memory | Bandwidth |
|---|---|
| NVMe SSD (PCIe 4.0) | up to about 7 GB/s |
| RAM in PCs and laptops (DDR4 / DDR5, two channels) | about 50–90 GB/s |
| Mac with M1 | about 68 GB/s |
| Mac with M4 | 120 GB/s |
| Mac with M1 Pro | 200 GB/s |
| Mac with M4 Max | up to 546 GB/s |
| Graphics card NVIDIA RTX 4090 | about 1,000 GB/s |

Sources: [Apple M1](https://en.wikipedia.org/wiki/Apple_M1), [Apple M1 Pro](https://www.apple.com/newsroom/2021/10/introducing-m1-pro-and-m1-max-the-most-powerful-chips-apple-has-ever-built/), [Apple M4 family](https://www.apple.com/newsroom/2024/10/apple-introduces-m4-pro-and-m4-max/), [NVIDIA RTX 4090](https://www.nvidia.com/en-us/geforce/graphics-cards/40-series/rtx-4090/). RAM values are calculated from the memory speed (e.g. DDR5-5600: 5,600 million transfers × 8 bytes × 2 channels ≈ 90 GB/s).

### Example

With the rule of thumb, an 8B model with Q4_K_M (4.6 GB) answers at most at:

| Model is read from | Tokens per second (upper limit) |
|---|---|
| NVMe SSD | about 1.5 |
| Mac with M1 | about 15 |
| RAM with DDR5 | about 20 |
| Mac with M1 Pro | about 45 |
| NVIDIA RTX 4090 | about 220 |

Real values are lower, because other work is added. The comparison still shows the order of magnitude: reading from the SSD is far too slow for working with an agent, a graphics card is much faster than RAM.

For models with experts, only the active parameters are read per token. A 30B-A3B model with Q4_K_M reads only about 2 GB per token and is therefore about twice as fast as the 8B model in this example, if it fits into memory (see [Active parameters](5-models.md#active-parameters)).
