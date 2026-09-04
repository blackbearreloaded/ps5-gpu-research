# Compute and AI findings

## Current status

General GPU compute and complete autoregressive transformer inference are
hardware-proven on firmware 6.02. The production experiment is a purpose-built
native runtime, not a port of ROCm, CUDA, or a full OpenCL implementation.

The runtime has exercised:

- storage-buffer input and output;
- FP32 arithmetic and reductions;
- large multi-workgroup dispatches;
- quantized matrix-vector projections;
- normalization, attention, activation, residual, and logits stages;
- persistent model weights and key/value caches;
- prompt prefill and autoregressive decode;
- model-specific tokenizers and prompt templates; and
- native UI streaming and model switching.

## Progression from canary to model

The investigation deliberately advanced through observable layers:

1. write distinct values from one compute workgroup into a caller-owned buffer;
2. scale the same operation to one million verified output words;
3. read CPU-populated input buffers and copy exact operands back;
4. execute an FP32 dot product and match a CPU oracle;
5. coalesce large matrix rows across workgroups and parallel reductions;
6. execute packed Q8 and Q4 projections;
7. load one real quantized model tensor and match all reference rows;
8. execute one complete transformer layer;
9. execute every model layer and select one reference token;
10. retain per-layer key/value state across autoregressive tokens;
11. tokenize prompts and stream decoded text; and
12. scale to multi-billion-parameter model profiles with resident weights.

Each layer retained a deterministic CPU or reference-runtime oracle before the
next was attempted.

## Compute execution model

```text
validated host/model data
  -> stable GPU-visible allocations
  -> typed storage-buffer descriptors
  -> target compute shader
  -> CPU-to-GPU visibility boundary
  -> ordered dispatch graph
  -> release/completion marker
  -> GPU-to-CPU visibility boundary
  -> exact tensor, token, or application oracle
```

The successful implementation uses a compact command graph with one submission
per generated token where possible. Dependent layer and logits dispatches stay
inside that graph and synchronize on the GPU.

## Foundational findings

### Resource typing is mandatory

Shader source alone is insufficient. Storage-buffer type, element stride,
access, and descriptor layout must survive the compiler and native object
boundaries. Missing resource metadata produced a shader that submitted and
completed while leaving the output untouched.

### CPU/GPU visibility must be explicit

Inputs written by the CPU require a visibility boundary before dispatch.
Likewise, a completion marker alone does not guarantee that CPU reads see the
latest shader output without the correct ownership transition.

### Compiler and dispatch wave modes must match

A wave-mode mismatch produced deterministic but incorrect arithmetic because
high vector registers aliased differently at execution. Matching dispatch mode
to compiled shader mode restored the exact FP32 result.

### Stable mappings are part of descriptor lifetime

Growing and remapping scratch storage after descriptors were published left
existing model buffers referencing obsolete mappings. Reserving the arena
before model execution and keeping its mapping fixed removed the fault.

### Tiny dispatches hide useful throughput

Early canaries were dominated by command submission and millisecond-scale
polling. Larger matrices and persistent command construction were required
before bandwidth or arithmetic comparisons became meaningful.

## Kernel findings

### Memory access and reductions

Mapping one output row to one lane caused adjacent lanes to read far-apart
weights. Assigning a workgroup to each row enabled contiguous loads and shared
parallel reduction, producing a large controlled speedup while retaining exact
output.

Increasing workgroup size alone did not guarantee improvement. A measured
256-thread fused-layer configuration outperformed otherwise equivalent 512-
and 1024-thread variants, motivating smaller ordered kernels rather than one
oversized kernel.

### Quantization

The model kernels support packed integer weights and activations with
model-specific scale handling.

Key result: converting Q4_0 block scales from FP16 to FP32 while preparing the
runtime image was much faster than repeatedly converting scales inside the
shader. The quantized nibbles remain unchanged; storage increases modestly from
0.5625 to 0.625 bytes per weight for that layout.

The best controlled large projection in this investigation matched every CPU
reference row and reached a conservative 59.336 GMAC/s lower bound. This number
is specific to that packed projection, dimensions, data reuse, timing method,
and firmware.

### Submission granularity

Submitting every transformer phase independently created a fixed queue cost
that dominated small models. Placing all layer phases and logits in one ordered
command sequence per token improved a 135M model path from roughly two tokens/s
to about 60 tokens/s in its controlled short-context test.

This does not imply that one giant dispatch is best. Separate kernels remain
useful for occupancy, resource lifetime, and synchronization; the gain comes
from ordering them in one submission.

## Transformer runtime

The complete runtime includes:

- packed model images arranged for direct GPU consumption;
- concurrent file reads into final GPU-visible storage;
- architecture-specific tensor metadata and kernels;
- model-wide decoder weights kept resident;
- per-layer key/value caches retained between tokens and turns;
- prompt-prefix reuse when conversation history is unchanged;
- batched prompt prefill, currently up to 256 positions per chunk in larger
  profiles;
- one ordered GPU graph per decode token;
- CPU tokenization, greedy token selection, text decoding, and UI; and
- architecture-aware unload/reload during model switching.

The CPU remains the application coordinator. The claim is that transformer
layers and logits execute on the GPU, not that every part of the application
runs there.

## Model coverage

| Profile | Proven result | Current boundary |
| --- | --- | --- |
| SmolLM2 135M | Full model, token-for-token oracle, 64-position sustained path around 60 tokens/s | Capability demonstrator; limited answer quality |
| SmolLM2 360M | Exact short and near-limit token oracles, native streamed UI | 512 positions; long serial prefill is not interactive |
| SmolLM2 1.7B | Exact reference sequences, multi-turn KV reuse, roughly 20 decode tokens/s | 512-position profile and model-specific kernels |
| SmolLM3 3B | Exact reference sequence and coherent native chat | Approximately 10.8-20 tokens/s depending context |
| Mistral 7B Q4_0 | Complete 32-layer GPU execution, exact oracle, coherent multi-turn chat | 4,096 positions; approximately 5.42 GiB mapped footprint |
| Qwen3.5 9B quantized | Complete hybrid recurrent/attention model, coherent response, switch to/from Mistral | Approximately 7.4 GiB GPU-visible footprint; model-specific path |

The 7B controlled 64-token test completed in 1.919 seconds, including 0.083
seconds for prompt prefill/first logits and 1.836 seconds for the remaining 63
decode steps, or about 34.3 decode tokens/s. A 4,095-token boundary prompt also
completed through position 4,095, but cold prefill at that size took 84.291
seconds.

The Qwen3.5 9B path uses quantized decoder and output projection formats and
supports up to 4,096 positions in the current runtime. Hardware validation
produced coherent output, then switched to Mistral and back to Qwen in the same
process while reproducing the expected Qwen first token.

## Loading and residency

Packing tensors into final runtime order made loading predictable and removed
per-tensor transformation from application startup. The 7B runtime image is
approximately 4.66 GB. Sixteen concurrent 16 MiB reads loaded it into its final
GPU-visible allocations in 1.455 seconds, followed by a 0.079-second cache
visibility step.

That runtime maps approximately 5.42 GiB:

- 4,224 MiB decoder weights;
- 1,104 MiB work and key/value storage;
- 224 MiB compact global tensors; and
- small alignment padding.

Later turns report no model reload. Model switching explicitly unloads one
architecture before preparing another, so the 7B and 9B models do not need to
fit simultaneously.

## Correctness method

Compute acceptance uses one or more of:

- exact dword or floating-point buffer comparison;
- maximum absolute error against a CPU tensor oracle;
- exact greedy token IDs against llama.cpp;
- decoded-text coherence for an end-to-end application path;
- stable model and tokenizer identities; and
- clean bounded title teardown and service health.

A coherent sentence alone is weaker than an exact token oracle. Both are kept
when available.

## Current limits

- Kernels and model packing are architecture-specific.
- Greedy selection is CPU-side; broader sampling policies are application work.
- Long-context prompt prefill remains materially slower than short decode.
- Peak theoretical throughput is not measured.
- The best projection benchmark must not be generalized to full-model speed.
- Multi-process compute sharing and preemption are not established.
- Most evidence is firmware-6.02 specific.
- Generative-audio work is not included because complete model-valid output has
  not yet met the same acceptance standard.

## Claim boundary

The evidence proves that the PS5 GPU can execute independently compiled compute
shaders, consume large caller-owned model allocations, run complete quantized
transformer graphs, retain KV state, and produce reference-matching tokens. It
does not establish a universal AI runtime, support for arbitrary GGUF models,
or a general-purpose compute API compatible with desktop ecosystems.
