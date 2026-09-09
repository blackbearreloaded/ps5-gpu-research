# PS5 GPU Research

Private, independent research into graphics and general-purpose GPU workloads
on PlayStation 5. The project documents a Mesa-based OpenGL 3.3 Core implementation,
a native compute path, and complete transformer inference executed on the GPU.

Graphics and evidence summaries updated September 8, 2026. Compute results
retain their previously recorded scope. OpenGL is a validation frontend for the
graphics findings, not a requirement for using the underlying GPU concepts.

This repository is deliberately separate from
[PS5 Hardware Video Decoding Research](https://github.com/blackbearreloaded/ps5-hardware-video-decoding-research).
Video decoding uses the console's media-decoder path. The work documented here
uses programmable GPU shaders for rendering and compute. The projects overlap
only where a decoded surface is sampled, converted, composited, or presented by
the GPU.

## Project status

| Area | Status |
| --- | --- |
| Primary validated firmware | PlayStation 5 firmware 6.02 |
| Native shader execution | Graphics and compute programs execute with deterministic CPU-visible results |
| Shader toolchain | Open-source GLSL/SPIR-V, Mesa NIR, and AMD compiler path adapted to the target GPU |
| OpenGL | Mesa-derived OpenGL 3.3 Core and GLSL 3.30 implementation; not certified |
| OpenGL API surface | All 344 OpenGL 3.3 Core commands exported and statically audited |
| OpenGL hardware coverage | Major Core 3.3 feature families have exact public-API hardware oracles |
| OpenGL validation | Historical frozen four-configuration campaign complete; later optimized binaries have separate, narrower evidence |
| Graphics performance | Small windowed scene at ~119.88 FPS across 1080p, 1440p and 4K; 128-cube scene at ~59.94 FPS at 1080p |
| Physical display negotiation | Later frozen apps verify native 1440p and 4K HDMI at 119.88 Hz on the tested connection; rendering and sink observations remain separate |
| Bounded graphics stability | Ten-minute ~59.90 FPS session and five native launch/exit cycles; not exhaustive recovery or memory validation |
| General compute | Storage-buffer reads/writes, floating-point reductions, quantized projections, synchronization, and large dispatches proven |
| Transformer inference | Complete 135M, 360M, 1.7B, 3B, 7B, and 9B-class model paths exercised on the GPU |
| Application integration | Offline chat UI supports resident models, KV-cache reuse, streaming responses, and model switching |

The historical September 7 campaign accounts for 9,886 cases across four
configurations: **37,404 Pass + 2,140 individually reviewed NotSupported =
39,544 accounted results**, not 39,544 passes. The runner has six disclosed
adaptations. This is project acceptance, not Khronos certification. Later
performance, game-derived and CI-built artifacts do not inherit that campaign.
See [evidence identities and limits](docs/evidence.md#graphics-evidence-identities).

## Research highlights

| Finding | Result |
| --- | --- |
| Shared graphics/compute foundation | Both paths use the same GPU-visible memory, shader compiler, descriptors, command submission, cache management, and completion model |
| Clean OpenGL architecture | Mesa provides API validation and state tracking; a dedicated Gallium backend translates normalized state to the native graphics interface |
| Real Core context | Mesa derives OpenGL 3.3 Core and GLSL 3.30 without version or extension overrides |
| Public API coverage | Buffers, textures, framebuffer objects, depth/stencil, blending, MSAA, geometry shaders, transform feedback, queries, synchronization, and instancing are hardware-proven in bounded slices |
| Shader portability | Runtime GLSL and offline SPIR-V reach target GPU machine code through open-source compiler components |
| Deterministic compute | GPU buffers, FP32 reductions, and quantized matrix projections match CPU reference results |
| Submission granularity | Combining dependent model phases into one ordered GPU command sequence removed most tiny-submit overhead |
| Graphics scheduling | Bounded batching, clear ordering and scoped cache maintenance substantially improve measured rendering without relaxing completion checks |
| Display-path limits | The same 4K app changed from 1080p120 to 4K120 after using a capable TV input; rendering dimensions and refresh-only overlays are not HDMI-resolution proof |
| Storage-path cost | A matched 1080p offscreen case improved from 14.10 to 19.98 FPS through CPU copy optimization; fast presentation is not proof of fast render-to-texture |
| Allocation lifetime | A game-derived candidate keeps persistent textures from displacing transient buffers; arena pressure is not equivalent to total GPU-memory exhaustion |
| Memory residency | Packed model weights and KV caches remain in GPU-visible direct memory between turns |
| Quantized inference | Q4/Q8 model layouts run directly; expanding small quantization scales during model preparation materially improved throughput |
| Model correctness | Multiple model profiles reproduced reference token sequences; larger models also generated coherent multi-turn responses |
| Practical 7B result | A controlled 64-token Mistral run completed in 1.919 seconds, about 34.3 decode tokens/s after preparation |
| Practical 9B result | A Qwen3.5 9B-class hybrid model generated coherent output and switched to/from the 7B model in one process |

## Proven data paths

### Graphics

```text
OpenGL application
  -> EGL fullscreen context
  -> Mesa API validation and state tracker
  -> PS5 Gallium driver
  -> open-source shader compiler
  -> native shader and resource objects
  -> graphics command submission
  -> render target / texture / readback
  -> VideoOut presentation or deterministic test receipt
```

### Compute

```text
Model or compute workload
  -> validated tensor and buffer layout
  -> SPIR-V / Mesa compiler path
  -> native compute shader object
  -> GPU-visible descriptors and direct memory
  -> ordered compute dispatches
  -> completion marker and cache visibility
  -> CPU oracle, token selection, or application output
```

The compute runtime is purpose-built for the console GPU. It is not a ROCm,
CUDA, or general OpenCL implementation.

## OpenGL 3.3 summary

The current graphics stack combines:

- EGL 1.4-style native fullscreen context and presentation integration;
- Mesa's OpenGL state tracker and public API implementation;
- a PS5 Gallium screen/context/resource backend;
- runtime GLSL 3.30 compilation through Mesa NIR and the AMD compiler path;
- native buffer, texture, render-target, depth/stencil, and shader resources;
- bounded command submission, completion, and readback; and
- a native application that runs the Khronos `KHR-GL33` package.

Hardware-proven feature families include:

- direct and indexed points, lines, strips, fans, and triangles;
- base vertex, primitive restart, instancing, attribute divisors, and packed or
  integer vertex data;
- normalized, sRGB, floating-point, integer, packed, depth, stencil, and
  compressed-texture fallback formats;
- 1D, 2D, 3D, rectangle, cube, multisample, and array textures;
- mipmaps, explicit LOD, filtering, wrapping, border color, swizzle, and
  seamless cube edges;
- framebuffer objects, multiple render targets, masks, scissor, viewport,
  depth/stencil tests, blending, dual-source blending, and 4x MSAA;
- vertex, fragment, and geometry shaders with GLSL 3.30 linkage;
- uniform buffers, transform feedback, occlusion/conditional queries, timers,
  and synchronization; and
- direct rendering into alternating presentation buffers.

See [OpenGL 3.3 findings](docs/opengl-3.3.md).

## General compute and AI summary

The compute work progressed from an observable storage-buffer canary to large
matrix operations and full autoregressive models. Important results include:

- caller-provided storage buffers can be read and written by compute shaders;
- CPU uploads become visible to the GPU through an explicit synchronization
  boundary, and GPU output becomes visible after bounded completion;
- the shader's compiled wave mode must match dispatch mode;
- cooperative contiguous loads and parallel reductions outperform one-lane,
  strided mappings;
- persistent command construction substantially reduces per-dispatch overhead;
- GPU-resident model weights and KV caches eliminate reloads on later turns;
- prompt prefill can batch positions while token decode remains autoregressive;
- Q4_0/Q4_1/Q6_K and packed Q8 paths are functional in model-specific kernels;
  and
- native model switching releases the previous architecture before preparing
  the next, avoiding simultaneous residency of both large models.

See [Compute and AI findings](docs/compute.md) and
[benchmarks](docs/benchmarks.md).

## Evidence boundary

Evidence labels are intentionally narrow:

- **Hardware-proven:** the exact workload ran on a console and matched its
  declared oracle.
- **Controlled:** deterministic data and a bounded application lifecycle were
  used, but broad product behavior is not inferred.
- **Source-derived:** behavior follows from open-source compiler or API code
  and has not necessarily been executed on the console.
- **Host-checked:** a build, link or host test passed; this is not GPU execution.
- **Owner-observed:** physical interaction was reported by the owner; its scope
  is distinct from deterministic numerical acceptance.
- **Inferred:** multiple observations support the conclusion, but a direct
  discriminator is still missing.
- **Pending:** implementation or evidence is incomplete.

Passing a targeted shader, model, or CTS case proves only that tested boundary.
It does not prove every format, model architecture, workload size, firmware, or
conformance requirement.

See [Evidence and limits](docs/evidence.md).

## Known limitations

- The historical complete campaign is not a full-matrix result for later
  optimized or CI-built binaries, and no Khronos certification is claimed.
- Eligible OpenGL work uses bounded deferred batching; native completion remains
  checked. Concurrent queues and shared contexts need separate validation.
- Several compatibility paths use CPU fallback, including selected texture
  decompression, blits, and resolves.
- OpenGL compatibility-profile fixed-function behavior is outside the current
  Core 3.3 target.
- Compute kernels are model-specific rather than a general ML compiler.
- Performance figures describe exact controlled workloads, not universal GPU
  throughput.
- Offscreen CPU layout copies remain a bottleneck. Ten-minute sessions and
  bounded recreation do not establish multi-hour, GPU-memory, suspend/resume or
  device-loss stability. Game-derived allocator/cache changes remain separate
  from the canonical graphics SDK at the reviewed source snapshot.
- Most current claims are firmware-6.02 claims. Selected earlier graphics
  primitives were also exercised elsewhere, but full-stack firmware parity is
  not claimed.
- Generative-audio experiments are excluded until a complete model-valid run
  and output acceptance are preserved.

## Documentation

| Document | Purpose |
| --- | --- |
| [Architecture](docs/architecture.md) | Shared GPU foundation and the graphics/compute split |
| [OpenGL 3.3](docs/opengl-3.3.md) | API architecture, feature coverage, CTS status, and limits |
| [Compute and AI](docs/compute.md) | Compute progression, transformer runtime, model coverage, and lessons |
| [Benchmarks](docs/benchmarks.md) | Controlled performance measurements and measurement boundaries |
| [Evidence](docs/evidence.md) | Confidence labels, claim matrix, firmware scope, and pending work |
| [Publication policy](PUBLICATION.md) | Material allowed in this research repository |

## Repository boundary

This repository contains independently written documentation. It intentionally
does not contain:

- platform SDK files, system modules, firmware images, or proprietary headers;
- decompiled code, instruction-by-instruction proprietary reconstructions, or
  symbol/offset databases;
- signing material, credentials, personal paths, network addresses, or raw
  console logs;
- exploit, access-control bypass, update avoidance, or privilege-escalation
  instructions;
- captured proprietary shaders, game assets, model weights, or third-party
  files without redistribution permission; or
- claims broader than the controlled evidence.

The implementation, model packaging, and large raw evidence remain in their
separately controlled development repositories.

## External projects and standards

| Project or reference | Role |
| --- | --- |
| [Mesa](https://www.mesa3d.org/) | OpenGL state tracker, Gallium infrastructure, NIR, and AMD compiler components |
| [Khronos OpenGL 3.3](https://registry.khronos.org/OpenGL/specs/gl/glspec33.core.pdf) | API and language requirements |
| [Khronos VK-GL-CTS](https://github.com/KhronosGroup/VK-GL-CTS) | OpenGL conformance test package |
| [SPIR-V](https://registry.khronos.org/SPIR-V/) | Portable shader intermediate representation used by compute experiments |
| [llama.cpp](https://github.com/ggml-org/llama.cpp) | Independent token and tensor reference oracle |
| [PS5 Hardware Video Decoding Research](https://github.com/blackbearreloaded/ps5-hardware-video-decoding-research) | Separate media-decoder and decoded-surface presentation research |

## License and attribution

Repository-authored material is licensed under GPL-3.0. See [LICENSE](LICENSE)
and [NOTICE.md](NOTICE.md).

This project was developed with assistance from OpenAI Codex. Project
maintainers reviewed and validated the resulting documentation.

PlayStation and PS5 are trademarks of Sony Interactive Entertainment. This
project is independent and is not affiliated with or endorsed by Sony.
