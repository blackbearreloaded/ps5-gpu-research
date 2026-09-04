# Evidence and limits

## Evidence method

Every accepted hardware finding used a bounded native application and a
declared oracle. The evidence chain records:

- source and dependency revisions;
- deterministic build and artifact identity;
- firmware and test configuration;
- exact input dimensions or model profile;
- expected output and comparison method;
- bounded submission and completion;
- application teardown and environment health; and
- a claim no broader than the test.

Raw captures and proprietary material remain outside this repository.

## Confidence labels

| Label | Meaning |
| --- | --- |
| Hardware-proven | Exact frozen workload ran and matched its oracle |
| Controlled | Hardware-proven under deterministic input, not necessarily sustained product use |
| Source-derived | Supported by public/open-source code inspection but not direct console execution |
| Inferred | Multiple observations support the conclusion; discriminator remains |
| Pending | Implementation or required evidence is incomplete |
| Unavailable through examined path | No usable route was found; no physical-hardware absence is inferred |

## Current claim matrix

| Claim | Grade | Proven boundary |
| --- | --- | --- |
| Target GPU executes independently compiled graphics shaders | Hardware-proven | Vertex, fragment, and geometry stages with exact pixel oracles |
| Target GPU executes independently compiled compute shaders | Hardware-proven | Buffer writes, reads, reductions, and large dispatches |
| Graphics and compute share compiler/memory/submission concepts | Hardware-proven architecture | Reused compiler backend, resource model, visibility, and completion principles |
| Mesa can derive OpenGL 3.3 Core and GLSL 3.30 | Hardware-proven | Current candidate and implemented Gallium caps/formats |
| OpenGL 3.3 public command surface exists | Static and build-proven | All 344 Core commands link through the consumer SDK |
| Major Core 3.3 feature families operate | Hardware-proven slices | Exact public-API tests listed in the OpenGL document |
| Full OpenGL 3.3 conformance | Pending | Complete 39,544-execution immutable campaign not finished |
| Quantized matrix operations match CPU references | Hardware-proven | Exact tested Q4/Q8 layouts and dimensions |
| Complete transformer layers execute on GPU | Hardware-proven | Normalization, attention, projections, activation, residual, and logits |
| Complete autoregressive models execute on GPU | Hardware-proven | Listed 135M through 9B-class profiles |
| Arbitrary GGUF models run without new code | Not claimed | Runtime is architecture/profile specific |
| Video codecs execute on shader compute units | Not claimed | Video decoding is a separate media path |
| General OpenCL/ROCm/CUDA compatibility | Not claimed | Purpose-built native runtime only |
| Cross-firmware equivalence | Pending | Primary integrated evidence is firmware 6.02 |

## OpenGL evidence summary

The OpenGL result is supported by:

- a clean Mesa/Gallium architecture;
- normal Mesa version derivation;
- all required Core command symbols in the SDK;
- exact buffer, texture, shader, draw, framebuffer, multisample, query,
  transform-feedback, and synchronization oracles;
- direct presentation with changing frame hashes;
- targeted official Khronos test passes;
- preserved functional failures that led to general fixes; and
- clean bounded lifecycle receipts.

The final gap is breadth, not a known missing high-level architecture: one
unchanged executable must complete every official case in all four required
configurations. New failures discovered there remain implementation work.

## Compute evidence summary

The compute result is supported by:

- distinct GPU-written storage-buffer values;
- exact large-buffer fill validation;
- exact CPU-populated operand readback;
- FP32 dot and matrix results;
- quantized large projections matching every CPU row;
- one real model tensor and one full decoder layer matching independent
  references;
- full-model greedy token IDs matching llama.cpp;
- bounded context and KV-cache tests;
- coherent native application responses; and
- model switching with explicit residency management.

Failed intermediate artifacts are not reused as positive evidence. Incorrect
output, timeout, GPU fault, process failure, soft lock, and kernel panic remain
separate classifications.

## Firmware scope

The integrated OpenGL Core candidate and AI runtime are primarily validated on
firmware 6.02. Selected earlier graphics primitives were exercised in another
controlled firmware environment, but this repository does not infer full
feature or performance parity.

Firmware interfaces and behavior can change. Every future claim should name
the exact firmware and reuse identical frozen bytes when comparing versions.

## Reproducibility boundary

This high-level repository is not the implementation repository. Reproducing a
hardware result also requires separately controlled:

- source trees and open-source dependency pins;
- a lawful native application build environment;
- model files obtained under their own licenses;
- deterministic packaging recipes;
- a console test protocol; and
- raw receipts kept outside Git.

No result should be reconstructed from prose alone and called an independent
reproduction.

## Pending work

1. Complete the official OpenGL 3.3 campaign on one immutable candidate.
2. Run representative external OpenGL 3.3 renderers through the installed SDK.
3. Characterize OpenGL performance only after correctness stabilizes.
4. Expand compute correctness to additional model architectures without
   weakening exact reference checks.
5. Add multi-process or concurrent-queue tests only with an explicit safety
   and lifecycle design.
6. Repeat the final integrated workloads on additional firmware using the same
   frozen artifacts.
7. Publish generative-audio findings only after complete, model-valid output
   and playback acceptance.

## Publication boundary

This repository preserves conclusions, not proprietary implementation.
Anything that would disclose system binaries, copied code, non-public SDK
material, offsets, raw command captures, credentials, signing material, or
access-control instructions is excluded. See [`PUBLICATION.md`](../PUBLICATION.md).
