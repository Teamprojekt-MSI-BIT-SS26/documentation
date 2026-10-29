# Models

This page describes what a model needs: memory, context and quantization, what matters for agent mode and where to find models. For an overview of all building blocks, see [Overview](1-overview.md).

!!! note "Not every model runs on every engine"
    New architectures and quantization formats are not supported by every engine and every chip. See [Engines and models](4-engines.md#engines-and-models).

## Proprietary and open-weight models

| | Proprietary | Open-weight |
|---|---|---|
| **Examples** | Claude, GPT, Gemini | Qwen, DeepSeek, Gemma, gpt-oss |
| **Access** | Only through the provider, by subscription or API | Download and run ourselves, or use through an API, e.g. [OpenRouter](https://openrouter.ai/models) |
| **Costs** | Subscription or price per token | Own hardware, or a price per token that is usually lower |
| **Data privacy** | Code goes to the provider | Stays with us if we run the model ourselves; goes to the provider if we use an API |

Open-weight models through an API are a cloud option without own hardware, usually cheaper than proprietary models.

## Parameters and memory

A model consists of parameters: numbers learned during training. The size is given in billions, e.g. "30B" for 30 billion parameters.

Memory for the model ≈ parameters × bits per parameter ÷ 8

- Original (16 bits = 2 bytes per parameter): a 30B model needs about 60 GB.
- Quantized to about 4 bits: the same model needs about 19 GB (see [Quantization](#quantization)).

For a fast answer, the whole model should fit into the memory of the GPU: VRAM on PCs, or the part of the unified memory the GPU may use on Macs (see [Hardware](6-hardware.md#memory)).

!!! important "The model is not the only thing in memory"
    **Total memory = model + [context](#context) + operating system and other programs**

## Context

The context is everything the model sees at once: our request, the code the tool sends along, the conversation so far and the answer. It is measured in tokens; one token is roughly ¾ of a word.

Agents need a large context, because they read whole files and keep the history of their steps. Ollama recommends at least 64K tokens for Codex CLI ([Ollama docs](https://docs.ollama.com/integrations/codex)).

The context needs its own memory, the **KV cache**. It grows with every token. Example for Qwen3-Coder-30B-A3B (about 96 KB per token at 16 bits):

| Context | KV cache (16 bits) | KV cache (8 bits) |
|---|---|---|
| 8K tokens | about 0.75 GB | about 0.4 GB |
| 32K tokens | about 3 GB | about 1.5 GB |
| 64K tokens | about 6 GB | about 3 GB |
| 256K tokens | about 24 GB | about 12 GB |

- The model card states the **maximum** context a model can handle. How much is actually reserved is set in the engine.
- The KV cache can be stored with 8 bits instead of 16 to halve its memory, with a small loss in quality (e.g. `--cache-type-k q8_0` in [llama.cpp](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)).
- The memory per token differs between models. It is calculated as 2 × layers × KV heads × head size × bytes per value; for Qwen3-Coder-30B-A3B: 2 × 48 × 4 × 128 × 2 bytes.

## Active parameters

Many current models are **mixture-of-experts (MoE)** models. They consist of many specialized parts, the experts. For every token, a router selects only a few of them.

The name shows both numbers: **30B-A3B** means 30 billion parameters in total, of which about 3 billion are active per token.

| | Decides | Why |
|---|---|---|
| **Total parameters** | Memory | All experts have to be loaded, because each token may need different ones. |
| **Active parameters** | Speed | Per token, only the weights of the active experts are read. |

Example: [Qwen3-Coder-30B-A3B](https://huggingface.co/Qwen/Qwen3-Coder-30B-A3B-Instruct) has 30.5 billion parameters, of which 3.3 billion are active (8 of 128 experts per token). It needs memory like a 30B model, but answers about as fast as a 3B model.

In models without experts (dense models), all parameters are active for every token.

## Offloading

If a model does not fit into VRAM, the engine can keep a part of it in RAM and compute that part on the CPU. How well this works depends on the model.

<!-- Offloading graphic, hidden for now
<svg viewBox="0 0 840 340" width="100%" role="img" aria-label="Offloading: a dense model reads its layers in RAM for every token, a mixture-of-experts model reads only a few experts from RAM per token" style="max-width:840px;font-size:13px" fill="none" stroke="currentColor">
<rect x="0" y="6" width="16" height="14" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="22" y="18" fill="currentColor" stroke="none">read for this token</text>
<rect x="170" y="6" width="16" height="14" rx="3" stroke-dasharray="3 3" opacity="0.6"/>
<text x="192" y="18" fill="currentColor" stroke="none">not read for this token</text>
<text x="0" y="50" fill="currentColor" stroke="none" font-weight="bold" font-size="15">Dense model</text>
<rect x="0" y="60" width="400" height="96" rx="6" opacity="0.8"/>
<rect x="440" y="60" width="400" height="96" rx="6" opacity="0.8" stroke-dasharray="8 4"/>
<text x="10" y="78" fill="currentColor" stroke="none" font-weight="bold">GPU memory (VRAM): fast</text>
<text x="450" y="78" fill="currentColor" stroke="none" font-weight="bold">Main memory (RAM): slow</text>
<line x1="404" y1="108" x2="436" y2="108" stroke-width="1.5"/>
<path d="M430,103 L436,108 L430,113" />
<rect x="14" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="76" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="138" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="200" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="262" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="324" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="14" y="142" fill="currentColor" stroke="none" font-size="12">Layers 1–6</text>
<rect x="454" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="516" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="578" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<rect x="640" y="92" width="50" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="454" y="142" fill="currentColor" stroke="none" font-size="12">Layers 7–10</text>
<text x="0" y="178" fill="currentColor" stroke="none">Every token passes through all layers, including those in RAM. The slow part delays every token.</text>
<text x="0" y="200" fill="currentColor" stroke="none" font-weight="bold" font-size="15">Mixture of experts</text>
<rect x="0" y="210" width="400" height="96" rx="6" opacity="0.8"/>
<rect x="440" y="210" width="400" height="96" rx="6" opacity="0.8" stroke-dasharray="8 4"/>
<text x="10" y="228" fill="currentColor" stroke="none" font-weight="bold">GPU memory (VRAM): fast</text>
<text x="450" y="228" fill="currentColor" stroke="none" font-weight="bold">Main memory (RAM): slow</text>
<line x1="404" y1="258" x2="436" y2="258" stroke-width="1.5"/>
<path d="M430,253 L436,258 L430,263" />
<rect x="14" y="242" width="80" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="14" y="292" fill="currentColor" stroke="none" font-size="12">Attention</text>
<rect x="106" y="242" width="80" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="106" y="292" fill="currentColor" stroke="none" font-size="12">Router</text>
<rect x="198" y="242" width="80" height="30" rx="3" fill="currentColor" fill-opacity="0.55"/>
<text x="198" y="292" fill="currentColor" stroke="none" font-size="12">Context</text>
<rect x="454" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="480" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="506" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="532" y="238" width="20" height="15" rx="2" fill="currentColor" fill-opacity="0.55"/>
<rect x="558" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="584" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="610" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="636" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="662" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="688" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="714" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="740" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="766" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="792" y="238" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="454" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="480" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="506" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="532" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="558" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="584" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="610" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="636" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="662" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="688" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="714" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="740" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="766" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="792" y="258" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="454" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="480" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="506" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="532" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="558" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="584" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="610" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="636" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="662" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="688" y="278" width="20" height="15" rx="2" fill="currentColor" fill-opacity="0.55"/>
<rect x="714" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="740" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="766" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<rect x="792" y="278" width="20" height="15" rx="2" stroke-dasharray="3 3" opacity="0.6"/>
<text x="0" y="328" fill="currentColor" stroke="none">Shared parts and context stay on the GPU. Of the many experts in RAM, each token reads only a few.</text>
</svg>
-->

| Model type | Effect of offloading | Why |
|---|---|---|
| **Dense** | Much slower | Every token needs all weights, so the slow part in RAM delays every token. |
| **Mixture of experts** | Still usable | The experts are moved to RAM, the parts used for every token stay on the GPU. Per token only a few experts are read from RAM. |

- llama.cpp moves the experts to RAM with `--cpu-moe` or `--n-cpu-moe N` ([llama.cpp server](https://github.com/ggml-org/llama.cpp/blob/master/tools/server/README.md)).
- Some engines even read parts of very large models from the SSD, e.g. Strata for DeepSeek. This needs a lot of free disk space and is slower.
- Macs with unified memory do not split between VRAM and RAM, but the GPU may only use a part of the memory (see [Hardware](6-hardware.md#memory)).

## Quantization

Normally each parameter is stored with 16 bits, which allows about 65,000 different values. Quantization rounds each parameter to fewer values: 256 with 8 bits, 16 with 4 bits, 8 with 3 bits. The model gets smaller and faster, but less precise. Quantization only matters if we run the model ourselves.

<svg viewBox="0 0 840 304" width="100%" role="img" aria-label="A parameter stored with 16, 8, 4 and 3 bits: with fewer bits the stored code gets shorter and the value is rounded more coarsely" style="max-width:840px;font-size:13px" fill="none" stroke="currentColor">
<text x="90" y="16" fill="currentColor" stroke="none" font-weight="bold">Stored bits</text>
<text x="341" y="16" fill="currentColor" stroke="none">−1</text>
<text x="468.5" y="16" fill="currentColor" stroke="none">0</text>
<text x="594" y="16" fill="currentColor" stroke="none">1</text>
<text x="625" y="16" fill="currentColor" stroke="none" font-weight="bold">Resulting value</text>
<line x1="512.4" y1="24" x2="512.4" y2="300" stroke-dasharray="4 3" opacity="0.5"/>
<text x="0" y="50" fill="currentColor" stroke="none" font-weight="bold">16 bits</text>
<text x="0" y="68" fill="currentColor" stroke="none" opacity="0.8">65,536 values</text>
<rect x="90" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="92.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="104" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="106.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="118" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="120.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="132" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="134.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="146" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="148.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="160" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="162.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="174" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="176.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="188" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="190.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="202" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="204.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="216" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="218.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="230" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="232.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="244" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="246.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="258" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="260.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="272" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="274.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="286" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="288.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="300" y="37" width="12" height="18" rx="2" opacity="0.7"/>
<text x="302.5" y="50.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<text x="90" y="70" fill="currentColor" stroke="none" opacity="0.8" font-size="12">16-bit floating point</text>
<line x1="345" y1="46" x2="600" y2="46" opacity="0.6"/>
<circle cx="512.4" cy="46" r="5" fill="currentColor"/>
<text x="625" y="50" fill="currentColor" stroke="none">0.312744 (original)</text>
<text x="0" y="116" fill="currentColor" stroke="none" font-weight="bold">8 bits</text>
<text x="0" y="134" fill="currentColor" stroke="none" opacity="0.8">256 values</text>
<rect x="90" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="92.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="104" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="106.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="118" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="120.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="132" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="134.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="146" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="148.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="160" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="162.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="174" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="176.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="188" y="103" width="12" height="18" rx="2" opacity="0.7"/>
<text x="190.5" y="116.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<text x="90" y="136" fill="currentColor" stroke="none" opacity="0.8" font-size="12">8-bit integer</text>
<line x1="345" y1="112" x2="600" y2="112" opacity="0.6"/>
<line x1="345.0" y1="106" x2="345.0" y2="118" stroke-width="0.4"/>
<line x1="346.0" y1="106" x2="346.0" y2="118" stroke-width="0.4"/>
<line x1="347.0" y1="106" x2="347.0" y2="118" stroke-width="0.4"/>
<line x1="348.0" y1="106" x2="348.0" y2="118" stroke-width="0.4"/>
<line x1="349.0" y1="106" x2="349.0" y2="118" stroke-width="0.4"/>
<line x1="350.0" y1="106" x2="350.0" y2="118" stroke-width="0.4"/>
<line x1="351.0" y1="106" x2="351.0" y2="118" stroke-width="0.4"/>
<line x1="352.0" y1="106" x2="352.0" y2="118" stroke-width="0.4"/>
<line x1="353.0" y1="106" x2="353.0" y2="118" stroke-width="0.4"/>
<line x1="354.0" y1="106" x2="354.0" y2="118" stroke-width="0.4"/>
<line x1="355.0" y1="106" x2="355.0" y2="118" stroke-width="0.4"/>
<line x1="356.0" y1="106" x2="356.0" y2="118" stroke-width="0.4"/>
<line x1="357.0" y1="106" x2="357.0" y2="118" stroke-width="0.4"/>
<line x1="358.0" y1="106" x2="358.0" y2="118" stroke-width="0.4"/>
<line x1="359.0" y1="106" x2="359.0" y2="118" stroke-width="0.4"/>
<line x1="360.0" y1="106" x2="360.0" y2="118" stroke-width="0.4"/>
<line x1="361.0" y1="106" x2="361.0" y2="118" stroke-width="0.4"/>
<line x1="362.0" y1="106" x2="362.0" y2="118" stroke-width="0.4"/>
<line x1="363.0" y1="106" x2="363.0" y2="118" stroke-width="0.4"/>
<line x1="364.0" y1="106" x2="364.0" y2="118" stroke-width="0.4"/>
<line x1="365.0" y1="106" x2="365.0" y2="118" stroke-width="0.4"/>
<line x1="366.0" y1="106" x2="366.0" y2="118" stroke-width="0.4"/>
<line x1="367.0" y1="106" x2="367.0" y2="118" stroke-width="0.4"/>
<line x1="368.0" y1="106" x2="368.0" y2="118" stroke-width="0.4"/>
<line x1="369.0" y1="106" x2="369.0" y2="118" stroke-width="0.4"/>
<line x1="370.0" y1="106" x2="370.0" y2="118" stroke-width="0.4"/>
<line x1="371.0" y1="106" x2="371.0" y2="118" stroke-width="0.4"/>
<line x1="372.0" y1="106" x2="372.0" y2="118" stroke-width="0.4"/>
<line x1="373.0" y1="106" x2="373.0" y2="118" stroke-width="0.4"/>
<line x1="374.0" y1="106" x2="374.0" y2="118" stroke-width="0.4"/>
<line x1="375.0" y1="106" x2="375.0" y2="118" stroke-width="0.4"/>
<line x1="376.0" y1="106" x2="376.0" y2="118" stroke-width="0.4"/>
<line x1="377.0" y1="106" x2="377.0" y2="118" stroke-width="0.4"/>
<line x1="378.0" y1="106" x2="378.0" y2="118" stroke-width="0.4"/>
<line x1="379.0" y1="106" x2="379.0" y2="118" stroke-width="0.4"/>
<line x1="380.0" y1="106" x2="380.0" y2="118" stroke-width="0.4"/>
<line x1="381.0" y1="106" x2="381.0" y2="118" stroke-width="0.4"/>
<line x1="382.0" y1="106" x2="382.0" y2="118" stroke-width="0.4"/>
<line x1="383.0" y1="106" x2="383.0" y2="118" stroke-width="0.4"/>
<line x1="384.0" y1="106" x2="384.0" y2="118" stroke-width="0.4"/>
<line x1="385.0" y1="106" x2="385.0" y2="118" stroke-width="0.4"/>
<line x1="386.0" y1="106" x2="386.0" y2="118" stroke-width="0.4"/>
<line x1="387.0" y1="106" x2="387.0" y2="118" stroke-width="0.4"/>
<line x1="388.0" y1="106" x2="388.0" y2="118" stroke-width="0.4"/>
<line x1="389.0" y1="106" x2="389.0" y2="118" stroke-width="0.4"/>
<line x1="390.0" y1="106" x2="390.0" y2="118" stroke-width="0.4"/>
<line x1="391.0" y1="106" x2="391.0" y2="118" stroke-width="0.4"/>
<line x1="392.0" y1="106" x2="392.0" y2="118" stroke-width="0.4"/>
<line x1="393.0" y1="106" x2="393.0" y2="118" stroke-width="0.4"/>
<line x1="394.0" y1="106" x2="394.0" y2="118" stroke-width="0.4"/>
<line x1="395.0" y1="106" x2="395.0" y2="118" stroke-width="0.4"/>
<line x1="396.0" y1="106" x2="396.0" y2="118" stroke-width="0.4"/>
<line x1="397.0" y1="106" x2="397.0" y2="118" stroke-width="0.4"/>
<line x1="398.0" y1="106" x2="398.0" y2="118" stroke-width="0.4"/>
<line x1="399.0" y1="106" x2="399.0" y2="118" stroke-width="0.4"/>
<line x1="400.0" y1="106" x2="400.0" y2="118" stroke-width="0.4"/>
<line x1="401.0" y1="106" x2="401.0" y2="118" stroke-width="0.4"/>
<line x1="402.0" y1="106" x2="402.0" y2="118" stroke-width="0.4"/>
<line x1="403.0" y1="106" x2="403.0" y2="118" stroke-width="0.4"/>
<line x1="404.0" y1="106" x2="404.0" y2="118" stroke-width="0.4"/>
<line x1="405.0" y1="106" x2="405.0" y2="118" stroke-width="0.4"/>
<line x1="406.0" y1="106" x2="406.0" y2="118" stroke-width="0.4"/>
<line x1="407.0" y1="106" x2="407.0" y2="118" stroke-width="0.4"/>
<line x1="408.0" y1="106" x2="408.0" y2="118" stroke-width="0.4"/>
<line x1="409.0" y1="106" x2="409.0" y2="118" stroke-width="0.4"/>
<line x1="410.0" y1="106" x2="410.0" y2="118" stroke-width="0.4"/>
<line x1="411.0" y1="106" x2="411.0" y2="118" stroke-width="0.4"/>
<line x1="412.0" y1="106" x2="412.0" y2="118" stroke-width="0.4"/>
<line x1="413.0" y1="106" x2="413.0" y2="118" stroke-width="0.4"/>
<line x1="414.0" y1="106" x2="414.0" y2="118" stroke-width="0.4"/>
<line x1="415.0" y1="106" x2="415.0" y2="118" stroke-width="0.4"/>
<line x1="416.0" y1="106" x2="416.0" y2="118" stroke-width="0.4"/>
<line x1="417.0" y1="106" x2="417.0" y2="118" stroke-width="0.4"/>
<line x1="418.0" y1="106" x2="418.0" y2="118" stroke-width="0.4"/>
<line x1="419.0" y1="106" x2="419.0" y2="118" stroke-width="0.4"/>
<line x1="420.0" y1="106" x2="420.0" y2="118" stroke-width="0.4"/>
<line x1="421.0" y1="106" x2="421.0" y2="118" stroke-width="0.4"/>
<line x1="422.0" y1="106" x2="422.0" y2="118" stroke-width="0.4"/>
<line x1="423.0" y1="106" x2="423.0" y2="118" stroke-width="0.4"/>
<line x1="424.0" y1="106" x2="424.0" y2="118" stroke-width="0.4"/>
<line x1="425.0" y1="106" x2="425.0" y2="118" stroke-width="0.4"/>
<line x1="426.0" y1="106" x2="426.0" y2="118" stroke-width="0.4"/>
<line x1="427.0" y1="106" x2="427.0" y2="118" stroke-width="0.4"/>
<line x1="428.0" y1="106" x2="428.0" y2="118" stroke-width="0.4"/>
<line x1="429.0" y1="106" x2="429.0" y2="118" stroke-width="0.4"/>
<line x1="430.0" y1="106" x2="430.0" y2="118" stroke-width="0.4"/>
<line x1="431.0" y1="106" x2="431.0" y2="118" stroke-width="0.4"/>
<line x1="432.0" y1="106" x2="432.0" y2="118" stroke-width="0.4"/>
<line x1="433.0" y1="106" x2="433.0" y2="118" stroke-width="0.4"/>
<line x1="434.0" y1="106" x2="434.0" y2="118" stroke-width="0.4"/>
<line x1="435.0" y1="106" x2="435.0" y2="118" stroke-width="0.4"/>
<line x1="436.0" y1="106" x2="436.0" y2="118" stroke-width="0.4"/>
<line x1="437.0" y1="106" x2="437.0" y2="118" stroke-width="0.4"/>
<line x1="438.0" y1="106" x2="438.0" y2="118" stroke-width="0.4"/>
<line x1="439.0" y1="106" x2="439.0" y2="118" stroke-width="0.4"/>
<line x1="440.0" y1="106" x2="440.0" y2="118" stroke-width="0.4"/>
<line x1="441.0" y1="106" x2="441.0" y2="118" stroke-width="0.4"/>
<line x1="442.0" y1="106" x2="442.0" y2="118" stroke-width="0.4"/>
<line x1="443.0" y1="106" x2="443.0" y2="118" stroke-width="0.4"/>
<line x1="444.0" y1="106" x2="444.0" y2="118" stroke-width="0.4"/>
<line x1="445.0" y1="106" x2="445.0" y2="118" stroke-width="0.4"/>
<line x1="446.0" y1="106" x2="446.0" y2="118" stroke-width="0.4"/>
<line x1="447.0" y1="106" x2="447.0" y2="118" stroke-width="0.4"/>
<line x1="448.0" y1="106" x2="448.0" y2="118" stroke-width="0.4"/>
<line x1="449.0" y1="106" x2="449.0" y2="118" stroke-width="0.4"/>
<line x1="450.0" y1="106" x2="450.0" y2="118" stroke-width="0.4"/>
<line x1="451.0" y1="106" x2="451.0" y2="118" stroke-width="0.4"/>
<line x1="452.0" y1="106" x2="452.0" y2="118" stroke-width="0.4"/>
<line x1="453.0" y1="106" x2="453.0" y2="118" stroke-width="0.4"/>
<line x1="454.0" y1="106" x2="454.0" y2="118" stroke-width="0.4"/>
<line x1="455.0" y1="106" x2="455.0" y2="118" stroke-width="0.4"/>
<line x1="456.0" y1="106" x2="456.0" y2="118" stroke-width="0.4"/>
<line x1="457.0" y1="106" x2="457.0" y2="118" stroke-width="0.4"/>
<line x1="458.0" y1="106" x2="458.0" y2="118" stroke-width="0.4"/>
<line x1="459.0" y1="106" x2="459.0" y2="118" stroke-width="0.4"/>
<line x1="460.0" y1="106" x2="460.0" y2="118" stroke-width="0.4"/>
<line x1="461.0" y1="106" x2="461.0" y2="118" stroke-width="0.4"/>
<line x1="462.0" y1="106" x2="462.0" y2="118" stroke-width="0.4"/>
<line x1="463.0" y1="106" x2="463.0" y2="118" stroke-width="0.4"/>
<line x1="464.0" y1="106" x2="464.0" y2="118" stroke-width="0.4"/>
<line x1="465.0" y1="106" x2="465.0" y2="118" stroke-width="0.4"/>
<line x1="466.0" y1="106" x2="466.0" y2="118" stroke-width="0.4"/>
<line x1="467.0" y1="106" x2="467.0" y2="118" stroke-width="0.4"/>
<line x1="468.0" y1="106" x2="468.0" y2="118" stroke-width="0.4"/>
<line x1="469.0" y1="106" x2="469.0" y2="118" stroke-width="0.4"/>
<line x1="470.0" y1="106" x2="470.0" y2="118" stroke-width="0.4"/>
<line x1="471.0" y1="106" x2="471.0" y2="118" stroke-width="0.4"/>
<line x1="472.0" y1="106" x2="472.0" y2="118" stroke-width="0.4"/>
<line x1="473.0" y1="106" x2="473.0" y2="118" stroke-width="0.4"/>
<line x1="474.0" y1="106" x2="474.0" y2="118" stroke-width="0.4"/>
<line x1="475.0" y1="106" x2="475.0" y2="118" stroke-width="0.4"/>
<line x1="476.0" y1="106" x2="476.0" y2="118" stroke-width="0.4"/>
<line x1="477.0" y1="106" x2="477.0" y2="118" stroke-width="0.4"/>
<line x1="478.0" y1="106" x2="478.0" y2="118" stroke-width="0.4"/>
<line x1="479.0" y1="106" x2="479.0" y2="118" stroke-width="0.4"/>
<line x1="480.0" y1="106" x2="480.0" y2="118" stroke-width="0.4"/>
<line x1="481.0" y1="106" x2="481.0" y2="118" stroke-width="0.4"/>
<line x1="482.0" y1="106" x2="482.0" y2="118" stroke-width="0.4"/>
<line x1="483.0" y1="106" x2="483.0" y2="118" stroke-width="0.4"/>
<line x1="484.0" y1="106" x2="484.0" y2="118" stroke-width="0.4"/>
<line x1="485.0" y1="106" x2="485.0" y2="118" stroke-width="0.4"/>
<line x1="486.0" y1="106" x2="486.0" y2="118" stroke-width="0.4"/>
<line x1="487.0" y1="106" x2="487.0" y2="118" stroke-width="0.4"/>
<line x1="488.0" y1="106" x2="488.0" y2="118" stroke-width="0.4"/>
<line x1="489.0" y1="106" x2="489.0" y2="118" stroke-width="0.4"/>
<line x1="490.0" y1="106" x2="490.0" y2="118" stroke-width="0.4"/>
<line x1="491.0" y1="106" x2="491.0" y2="118" stroke-width="0.4"/>
<line x1="492.0" y1="106" x2="492.0" y2="118" stroke-width="0.4"/>
<line x1="493.0" y1="106" x2="493.0" y2="118" stroke-width="0.4"/>
<line x1="494.0" y1="106" x2="494.0" y2="118" stroke-width="0.4"/>
<line x1="495.0" y1="106" x2="495.0" y2="118" stroke-width="0.4"/>
<line x1="496.0" y1="106" x2="496.0" y2="118" stroke-width="0.4"/>
<line x1="497.0" y1="106" x2="497.0" y2="118" stroke-width="0.4"/>
<line x1="498.0" y1="106" x2="498.0" y2="118" stroke-width="0.4"/>
<line x1="499.0" y1="106" x2="499.0" y2="118" stroke-width="0.4"/>
<line x1="500.0" y1="106" x2="500.0" y2="118" stroke-width="0.4"/>
<line x1="501.0" y1="106" x2="501.0" y2="118" stroke-width="0.4"/>
<line x1="502.0" y1="106" x2="502.0" y2="118" stroke-width="0.4"/>
<line x1="503.0" y1="106" x2="503.0" y2="118" stroke-width="0.4"/>
<line x1="504.0" y1="106" x2="504.0" y2="118" stroke-width="0.4"/>
<line x1="505.0" y1="106" x2="505.0" y2="118" stroke-width="0.4"/>
<line x1="506.0" y1="106" x2="506.0" y2="118" stroke-width="0.4"/>
<line x1="507.0" y1="106" x2="507.0" y2="118" stroke-width="0.4"/>
<line x1="508.0" y1="106" x2="508.0" y2="118" stroke-width="0.4"/>
<line x1="509.0" y1="106" x2="509.0" y2="118" stroke-width="0.4"/>
<line x1="510.0" y1="106" x2="510.0" y2="118" stroke-width="0.4"/>
<line x1="511.0" y1="106" x2="511.0" y2="118" stroke-width="0.4"/>
<line x1="512.0" y1="106" x2="512.0" y2="118" stroke-width="0.4"/>
<line x1="513.0" y1="106" x2="513.0" y2="118" stroke-width="0.4"/>
<line x1="514.0" y1="106" x2="514.0" y2="118" stroke-width="0.4"/>
<line x1="515.0" y1="106" x2="515.0" y2="118" stroke-width="0.4"/>
<line x1="516.0" y1="106" x2="516.0" y2="118" stroke-width="0.4"/>
<line x1="517.0" y1="106" x2="517.0" y2="118" stroke-width="0.4"/>
<line x1="518.0" y1="106" x2="518.0" y2="118" stroke-width="0.4"/>
<line x1="519.0" y1="106" x2="519.0" y2="118" stroke-width="0.4"/>
<line x1="520.0" y1="106" x2="520.0" y2="118" stroke-width="0.4"/>
<line x1="521.0" y1="106" x2="521.0" y2="118" stroke-width="0.4"/>
<line x1="522.0" y1="106" x2="522.0" y2="118" stroke-width="0.4"/>
<line x1="523.0" y1="106" x2="523.0" y2="118" stroke-width="0.4"/>
<line x1="524.0" y1="106" x2="524.0" y2="118" stroke-width="0.4"/>
<line x1="525.0" y1="106" x2="525.0" y2="118" stroke-width="0.4"/>
<line x1="526.0" y1="106" x2="526.0" y2="118" stroke-width="0.4"/>
<line x1="527.0" y1="106" x2="527.0" y2="118" stroke-width="0.4"/>
<line x1="528.0" y1="106" x2="528.0" y2="118" stroke-width="0.4"/>
<line x1="529.0" y1="106" x2="529.0" y2="118" stroke-width="0.4"/>
<line x1="530.0" y1="106" x2="530.0" y2="118" stroke-width="0.4"/>
<line x1="531.0" y1="106" x2="531.0" y2="118" stroke-width="0.4"/>
<line x1="532.0" y1="106" x2="532.0" y2="118" stroke-width="0.4"/>
<line x1="533.0" y1="106" x2="533.0" y2="118" stroke-width="0.4"/>
<line x1="534.0" y1="106" x2="534.0" y2="118" stroke-width="0.4"/>
<line x1="535.0" y1="106" x2="535.0" y2="118" stroke-width="0.4"/>
<line x1="536.0" y1="106" x2="536.0" y2="118" stroke-width="0.4"/>
<line x1="537.0" y1="106" x2="537.0" y2="118" stroke-width="0.4"/>
<line x1="538.0" y1="106" x2="538.0" y2="118" stroke-width="0.4"/>
<line x1="539.0" y1="106" x2="539.0" y2="118" stroke-width="0.4"/>
<line x1="540.0" y1="106" x2="540.0" y2="118" stroke-width="0.4"/>
<line x1="541.0" y1="106" x2="541.0" y2="118" stroke-width="0.4"/>
<line x1="542.0" y1="106" x2="542.0" y2="118" stroke-width="0.4"/>
<line x1="543.0" y1="106" x2="543.0" y2="118" stroke-width="0.4"/>
<line x1="544.0" y1="106" x2="544.0" y2="118" stroke-width="0.4"/>
<line x1="545.0" y1="106" x2="545.0" y2="118" stroke-width="0.4"/>
<line x1="546.0" y1="106" x2="546.0" y2="118" stroke-width="0.4"/>
<line x1="547.0" y1="106" x2="547.0" y2="118" stroke-width="0.4"/>
<line x1="548.0" y1="106" x2="548.0" y2="118" stroke-width="0.4"/>
<line x1="549.0" y1="106" x2="549.0" y2="118" stroke-width="0.4"/>
<line x1="550.0" y1="106" x2="550.0" y2="118" stroke-width="0.4"/>
<line x1="551.0" y1="106" x2="551.0" y2="118" stroke-width="0.4"/>
<line x1="552.0" y1="106" x2="552.0" y2="118" stroke-width="0.4"/>
<line x1="553.0" y1="106" x2="553.0" y2="118" stroke-width="0.4"/>
<line x1="554.0" y1="106" x2="554.0" y2="118" stroke-width="0.4"/>
<line x1="555.0" y1="106" x2="555.0" y2="118" stroke-width="0.4"/>
<line x1="556.0" y1="106" x2="556.0" y2="118" stroke-width="0.4"/>
<line x1="557.0" y1="106" x2="557.0" y2="118" stroke-width="0.4"/>
<line x1="558.0" y1="106" x2="558.0" y2="118" stroke-width="0.4"/>
<line x1="559.0" y1="106" x2="559.0" y2="118" stroke-width="0.4"/>
<line x1="560.0" y1="106" x2="560.0" y2="118" stroke-width="0.4"/>
<line x1="561.0" y1="106" x2="561.0" y2="118" stroke-width="0.4"/>
<line x1="562.0" y1="106" x2="562.0" y2="118" stroke-width="0.4"/>
<line x1="563.0" y1="106" x2="563.0" y2="118" stroke-width="0.4"/>
<line x1="564.0" y1="106" x2="564.0" y2="118" stroke-width="0.4"/>
<line x1="565.0" y1="106" x2="565.0" y2="118" stroke-width="0.4"/>
<line x1="566.0" y1="106" x2="566.0" y2="118" stroke-width="0.4"/>
<line x1="567.0" y1="106" x2="567.0" y2="118" stroke-width="0.4"/>
<line x1="568.0" y1="106" x2="568.0" y2="118" stroke-width="0.4"/>
<line x1="569.0" y1="106" x2="569.0" y2="118" stroke-width="0.4"/>
<line x1="570.0" y1="106" x2="570.0" y2="118" stroke-width="0.4"/>
<line x1="571.0" y1="106" x2="571.0" y2="118" stroke-width="0.4"/>
<line x1="572.0" y1="106" x2="572.0" y2="118" stroke-width="0.4"/>
<line x1="573.0" y1="106" x2="573.0" y2="118" stroke-width="0.4"/>
<line x1="574.0" y1="106" x2="574.0" y2="118" stroke-width="0.4"/>
<line x1="575.0" y1="106" x2="575.0" y2="118" stroke-width="0.4"/>
<line x1="576.0" y1="106" x2="576.0" y2="118" stroke-width="0.4"/>
<line x1="577.0" y1="106" x2="577.0" y2="118" stroke-width="0.4"/>
<line x1="578.0" y1="106" x2="578.0" y2="118" stroke-width="0.4"/>
<line x1="579.0" y1="106" x2="579.0" y2="118" stroke-width="0.4"/>
<line x1="580.0" y1="106" x2="580.0" y2="118" stroke-width="0.4"/>
<line x1="581.0" y1="106" x2="581.0" y2="118" stroke-width="0.4"/>
<line x1="582.0" y1="106" x2="582.0" y2="118" stroke-width="0.4"/>
<line x1="583.0" y1="106" x2="583.0" y2="118" stroke-width="0.4"/>
<line x1="584.0" y1="106" x2="584.0" y2="118" stroke-width="0.4"/>
<line x1="585.0" y1="106" x2="585.0" y2="118" stroke-width="0.4"/>
<line x1="586.0" y1="106" x2="586.0" y2="118" stroke-width="0.4"/>
<line x1="587.0" y1="106" x2="587.0" y2="118" stroke-width="0.4"/>
<line x1="588.0" y1="106" x2="588.0" y2="118" stroke-width="0.4"/>
<line x1="589.0" y1="106" x2="589.0" y2="118" stroke-width="0.4"/>
<line x1="590.0" y1="106" x2="590.0" y2="118" stroke-width="0.4"/>
<line x1="591.0" y1="106" x2="591.0" y2="118" stroke-width="0.4"/>
<line x1="592.0" y1="106" x2="592.0" y2="118" stroke-width="0.4"/>
<line x1="593.0" y1="106" x2="593.0" y2="118" stroke-width="0.4"/>
<line x1="594.0" y1="106" x2="594.0" y2="118" stroke-width="0.4"/>
<line x1="595.0" y1="106" x2="595.0" y2="118" stroke-width="0.4"/>
<line x1="596.0" y1="106" x2="596.0" y2="118" stroke-width="0.4"/>
<line x1="597.0" y1="106" x2="597.0" y2="118" stroke-width="0.4"/>
<line x1="598.0" y1="106" x2="598.0" y2="118" stroke-width="0.4"/>
<line x1="599.0" y1="106" x2="599.0" y2="118" stroke-width="0.4"/>
<line x1="600.0" y1="106" x2="600.0" y2="118" stroke-width="0.4"/>
<circle cx="512.0" cy="112" r="5" fill="currentColor"/>
<text x="625" y="116" fill="currentColor" stroke="none">0.309804 (error 0.003)</text>
<text x="0" y="182" fill="currentColor" stroke="none" font-weight="bold">4 bits</text>
<text x="0" y="200" fill="currentColor" stroke="none" opacity="0.8">16 values</text>
<rect x="90" y="169" width="12" height="18" rx="2" opacity="0.7"/>
<text x="92.5" y="182.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="104" y="169" width="12" height="18" rx="2" opacity="0.7"/>
<text x="106.5" y="182.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="118" y="169" width="12" height="18" rx="2" opacity="0.7"/>
<text x="120.5" y="182.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="132" y="169" width="12" height="18" rx="2" opacity="0.7"/>
<text x="134.5" y="182.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<text x="90" y="202" fill="currentColor" stroke="none" opacity="0.8" font-size="12">4-bit integer</text>
<line x1="345" y1="178" x2="600" y2="178" opacity="0.6"/>
<line x1="345.0" y1="172" x2="345.0" y2="184" stroke-width="1"/>
<line x1="362.0" y1="172" x2="362.0" y2="184" stroke-width="1"/>
<line x1="379.0" y1="172" x2="379.0" y2="184" stroke-width="1"/>
<line x1="396.0" y1="172" x2="396.0" y2="184" stroke-width="1"/>
<line x1="413.0" y1="172" x2="413.0" y2="184" stroke-width="1"/>
<line x1="430.0" y1="172" x2="430.0" y2="184" stroke-width="1"/>
<line x1="447.0" y1="172" x2="447.0" y2="184" stroke-width="1"/>
<line x1="464.0" y1="172" x2="464.0" y2="184" stroke-width="1"/>
<line x1="481.0" y1="172" x2="481.0" y2="184" stroke-width="1"/>
<line x1="498.0" y1="172" x2="498.0" y2="184" stroke-width="1"/>
<line x1="515.0" y1="172" x2="515.0" y2="184" stroke-width="1"/>
<line x1="532.0" y1="172" x2="532.0" y2="184" stroke-width="1"/>
<line x1="549.0" y1="172" x2="549.0" y2="184" stroke-width="1"/>
<line x1="566.0" y1="172" x2="566.0" y2="184" stroke-width="1"/>
<line x1="583.0" y1="172" x2="583.0" y2="184" stroke-width="1"/>
<line x1="600.0" y1="172" x2="600.0" y2="184" stroke-width="1"/>
<circle cx="515.0" cy="178" r="5" fill="currentColor"/>
<text x="625" y="182" fill="currentColor" stroke="none">0.333333 (error 0.021)</text>
<text x="0" y="248" fill="currentColor" stroke="none" font-weight="bold">3 bits</text>
<text x="0" y="266" fill="currentColor" stroke="none" opacity="0.8">8 values</text>
<rect x="90" y="235" width="12" height="18" rx="2" opacity="0.7"/>
<text x="92.5" y="248.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<rect x="104" y="235" width="12" height="18" rx="2" opacity="0.7"/>
<text x="106.5" y="248.5" fill="currentColor" stroke="none" font-family="monospace">0</text>
<rect x="118" y="235" width="12" height="18" rx="2" opacity="0.7"/>
<text x="120.5" y="248.5" fill="currentColor" stroke="none" font-family="monospace">1</text>
<text x="90" y="268" fill="currentColor" stroke="none" opacity="0.8" font-size="12">3-bit integer</text>
<line x1="345" y1="244" x2="600" y2="244" opacity="0.6"/>
<line x1="345.0" y1="238" x2="345.0" y2="250" stroke-width="1"/>
<line x1="381.4" y1="238" x2="381.4" y2="250" stroke-width="1"/>
<line x1="417.9" y1="238" x2="417.9" y2="250" stroke-width="1"/>
<line x1="454.3" y1="238" x2="454.3" y2="250" stroke-width="1"/>
<line x1="490.7" y1="238" x2="490.7" y2="250" stroke-width="1"/>
<line x1="527.1" y1="238" x2="527.1" y2="250" stroke-width="1"/>
<line x1="563.6" y1="238" x2="563.6" y2="250" stroke-width="1"/>
<line x1="600.0" y1="238" x2="600.0" y2="250" stroke-width="1"/>
<circle cx="527.1" cy="244" r="5" fill="currentColor"/>
<text x="625" y="248" fill="currentColor" stroke="none">0.428571 (error 0.116)</text>
</svg>

Example: a parameter with the value 0.3127, simplified to the range −1 to 1. With 16 bits it is stored as a floating-point number. With fewer bits, it is stored as an integer: the number of the nearest level, e.g. `101` for level 5 of the 8 levels possible with 3 bits. Each tick on the line is a level that can be stored; the dot is where the parameter ends up. With fewer bits, the levels are further apart and the rounding error grows. Real quantization works in small blocks with their own range, so the errors are smaller, but the principle is the same.

??? info "Why an integer and no longer a floating-point number?"
    A 16-bit floating-point number stores its own order of magnitude in every value:

    ```
    0      01101      0100000001
    │      │          └ mantissa (10 bits): the digits → 1.2509…
    │      └ exponent (5 bits): the order of magnitude → × 2⁻²
    └ sign (1 bit): positive
                               1.2509… × 2⁻² = 0.312744
    ```

    Quantization splits this work. The parameters are divided into small blocks, e.g. 32 parameters each. Each block stores one shared **scale**, which is itself a floating-point number. Each parameter then only stores the number of its level as a small integer:

    **value = scale of the block × level (integer)**, sometimes plus an offset

    The order of magnitude is stored once per block instead of in every parameter, which saves most of the bits. With 3 bits, a floating-point number would not make sense, because sign, exponent and digits do not fit into 3 bits; an integer from 0 to 7 does.

    In the graphic above, the scale is simplified to the fixed range −1 to 1.

    There are also very small floating-point formats such as FP8 or FP4 (e.g. NVFP4 on newer NVIDIA GPUs), which also use shared scales per block. The GGUF levels in the table below (Q8_0, Q4_K_M …), used by llama.cpp, Ollama and LM Studio, store integers with a scale.

The table shows the GGUF levels of llama.cpp for an 8B model (Llama 3). The quality loss is measured as the increase in **perplexity**: how uncertain the model is when predicting text. Higher means worse.

| Level | Bits per parameter | Size of an 8B model | Quality loss (perplexity increase) |
|---|---|---|---|
| F16 | 16 | about 15 GB | reference |
| Q8_0 | 8.5 | 7.96 GB | +0.003 |
| Q6_K | 6.6 | 6.14 GB | +0.02 |
| Q5_K_M | 5.7 | 5.33 GB | +0.06 |
| Q4_K_M | 4.9 | 4.58 GB | +0.18 |
| Q3_K_M | 4.0 | 3.74 GB | +0.66 |
| Q2_K | 3.2 | 2.96 GB | +3.52 |

Sources: [llama.cpp quantize](https://github.com/ggml-org/llama.cpp/blob/master/tools/quantize/README.md) and the level descriptions in its source code.

### From 4 to 3 bits

- **Memory:** Q4_K_M → Q3_K_M saves about 18 % (4.58 → 3.74 GB).
- **Quality:** The loss grows from +0.18 to +0.66, almost four times as much. Below 4 bits, quality drops much faster than between 8 and 4 bits. Q2_K loses about 20 times as much as Q4_K_M.
- **Larger model or more bits?** A larger model at 4 bits is usually better than a smaller one at more bits. A study found 4-bit 7B code models to be better than smaller models without quantization ([Lua study](https://arxiv.org/abs/2410.14766)).
- **Code needs precision:** Perplexity measures general text. Code and tool calls have to be exact, so a quantized model should be tested with our shared test task.

Q4_K_M is the usual choice for local use.

### Names of the levels

| Part | Meaning |
|---|---|
| **Q4, Q3 …** | About how many bits per parameter |
| **K** | K-quants: the model is quantized in small blocks, and important parts are stored more precisely |
| **S, M, L** | Small, medium, large: how many parts get more bits; L is more precise and larger |
| **IQ** | I-quants: a newer method with better quality at 2–3 bits |
| **4bit, 8bit** | Naming of MLX models |

## What a model needs for agent mode

- **Tool calling:** The model has to call tools such as "edit file" or "run command" reliably. The Ollama library marks such models with "tools"; the [Berkeley Function Calling Leaderboard](https://gorilla.cs.berkeley.edu/leaderboard.html) compares them.
- **Context:** at least about 64K tokens, see [Context](#context).
- **Coding quality:** The [Aider leaderboards](https://aider.chat/docs/leaderboards/) compare models when editing code, including costs; [SWE-bench](https://www.swebench.com) uses real GitHub issues. Benchmarks are only a guide; our shared test task decides.
- **Thinking models** think before they answer. Results are often better, but they produce many more tokens, which costs money with an API and time locally.

## Naming of model variants

| Part of the name | Meaning |
|---|---|
| **Coder** | Trained especially for programming |
| **Instruct**, **it** | Trained to follow instructions in a chat. "Base" models are not, and are not suited for us. |
| **Thinking**, **Reasoning** | Thinks before answering, see above |
| **Flash** | Fast and cheap to run. Not necessarily small: DeepSeek-V4.1-Flash and Qwen3.8-Flash-Next are large models. |
| **Mini**, **Small**, **Nano** | Smaller variant: faster, but weaker |
| **30B**, **80B** | Total parameters in billions, see [Parameters and memory](#parameters-and-memory) |
| **A3B** | Active parameters per token, see [Active parameters](#active-parameters) |
| **E4B** | Effectively about 4 billion parameters; small models for laptops |
| **MTP** | Multi-token prediction: predicts several tokens at once to answer faster |
| **Q4_K_M**, **4bit** | Quantization level, see [Names of the levels](#names-of-the-levels) |
| **GGUF**, **MLX** | Format of the files, see [Engines and models](4-engines.md#engines-and-models) |

## Finding models

- **Model card:** Every model on Hugging Face has a page with parameters, context length, licence and recommended settings ([model cards](https://huggingface.co/docs/hub/model-cards)).
- **Hardware check:** After entering our hardware once, Hugging Face shows for every GGUF or MLX model which quantization fits ([Hardware compatibility](https://huggingface.co/docs/hub/hardware)).
- **Trustworthy sources:** Prefer the official accounts of the developers (e.g. [Qwen](https://huggingface.co/Qwen)) and well-known quantizers such as [unsloth](https://huggingface.co/unsloth), [bartowski](https://huggingface.co/bartowski), [ggml-org](https://huggingface.co/ggml-org) or [mlx-community](https://huggingface.co/mlx-community). Variants named "uncensored", "abliterated" or merged models come from third parties and are not checked.

### Popular models (October 2026)

The following open-weight model families are among the most popular in the Ollama library and on Hugging Face (GGUF and MLX):

- **Qwen** (Alibaba): Qwen3.8 (e.g. 27B, Flash-Next), Qwen3.6-35B-A3B and the coding models Qwen3-Coder (30B-A3B) and Qwen3-Coder-Next
- **DeepSeek**: DeepSeek-V4.1-Flash
- **Gemma 4** (Google): up to 31B
- **gpt-oss** (OpenAI): 20B and 120B
- **Nemotron 3.5** (NVIDIA): small models, e.g. for laptops
- **MiniMax-M3.1-Flash**

Popularity changes quickly. Current lists:

- [Ollama library, sorted by popularity](https://ollama.com/library?sort=popular)
- [Hugging Face, trending GGUF models](https://huggingface.co/models?library=gguf&sort=trending)
- [Hugging Face, trending MLX models](https://huggingface.co/models?author=mlx-community&sort=trending)
