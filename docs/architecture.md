# GPU architecture findings

## Scope

This document describes the reusable architecture established by the graphics
and compute investigations. It intentionally omits proprietary implementation
details, raw command encodings, system-library internals, and platform-specific
addresses.

The central finding is that raster graphics and general compute can share most
of their low-level infrastructure while retaining different public frontends.

## Shared foundation

```text
                  +-----------------------------+
                  | Application                 |
                  +---------------+-------------+
                                  |
              +-------------------+-------------------+
              |                                       |
     +--------v---------+                    +--------v---------+
     | EGL / OpenGL     |                    | Compute runtime  |
     | Mesa state track |                    | Model scheduler  |
     +--------+---------+                    +--------+---------+
              |                                       |
              +-------------------+-------------------+
                                  |
                    +-------------v-------------+
                    | Shader compiler pipeline  |
                    | GLSL/SPIR-V -> NIR -> ISA |
                    +-------------+-------------+
                                  |
                    +-------------v-------------+
                    | Native GPU bridge         |
                    | shaders, resources, DCBs  |
                    +-------------+-------------+
                                  |
                    +-------------v-------------+
                    | GPU-visible direct memory |
                    | queues, barriers, markers |
                    +-------------+-------------+
                                  |
                     +------------v------------+
                     | PS5 GPU                 |
                     +------+-----------+------+
                            |           |
                      render/readback  compute output
```

The shared layer owns:

- target shader compilation and validation;
- shader metadata and native object creation;
- GPU-visible allocations and validated descriptors;
- command-buffer construction and bounded submission;
- CPU-to-GPU and GPU-to-CPU visibility;
- completion markers, waits, and resource lifetime; and
- deterministic diagnostics suitable for offline inspection.

OpenGL adds API validation, state tracking, raster state, render targets, and
presentation. Compute adds tensor layouts, dispatch scheduling, operator
kernels, and model state.

## Shader pipeline

The successful clean pipeline uses open-source components:

```text
GLSL or compute source
  -> standards-based frontend
  -> SPIR-V or Mesa NIR
  -> target-aware NIR lowering
  -> AMD compiler backend
  -> validated GFX10.3 machine code and metadata
  -> native shader object
```

Important conclusions:

- Graphics and compute need different stage metadata but can reuse compiler
  initialization, target information, validation, and packaging code.
- Resource declarations must survive every compiler boundary. A shader with a
  missing storage-buffer layout can compile while producing no useful memory
  access.
- Compiler resource limits must match the target architecture. Overstated
  register availability can turn ordinary registers into reserved hardware
  state.
- Compiled wave mode and dispatch mode must agree. A mismatch can produce
  plausible but incorrect arithmetic rather than a clean error.
- Shader creation should reject unsupported metadata or resource layouts
  before submission.

## Memory model

Both projects use caller-owned, GPU-visible direct memory for large resources.
The validated model is:

1. allocate with the required size and alignment;
2. retain one stable mapping while any descriptor points into it;
3. write or stream data into its final layout;
4. establish CPU-to-GPU visibility before dispatch or draw;
5. keep referenced storage alive until the queue completes;
6. establish GPU-to-CPU visibility before readback; and
7. release resources only after bounded completion and application teardown.

A key compute failure came from growing and remapping a scratch arena while
existing resources still pointed at the old mapping. Reserving the required
arena before publishing descriptors removes that class of fault.

OpenGL resources additionally track:

- format and numeric interpretation;
- dimensions, levels, layers, and samples;
- row, slice, and layer strides;
- linear or tiled storage;
- render, sampling, transfer, and presentation usage; and
- staging or fallback storage when a native path is unavailable.

Non-power-of-two mip chains require the hardware's physical storage rules,
which can differ from the logical OpenGL mip dimensions. The proven fix uses
ceiling division for physical linear mip extents while retaining the logical
floor dimensions exposed through OpenGL.

## Resource descriptors

Descriptors are treated as typed values, not opaque byte arrays. A descriptor
builder validates:

- GPU address and allocation bounds;
- format and component interpretation;
- dimensions, pitch, levels, layers, and samples;
- read/write intent;
- buffer element size and stride; and
- compatibility with the compiled shader declaration.

This model proved sufficient for graphics textures, render targets, uniform
buffers, storage buffers, model tensors, and completion labels.

## Command submission and synchronization

The safe common sequence is:

```text
validate resources
  -> make CPU uploads visible
  -> bind pipeline and descriptors
  -> emit ordered draws or dispatches
  -> emit release/completion marker
  -> submit once
  -> wait with a finite bound
  -> invalidate/read result when required
  -> retire referenced resources
```

Findings:

- A successful submission call is not completion evidence.
- Queue progress and a completion marker must be observed independently.
- Tiny submissions are dominated by fixed submission and polling overhead.
- Combining dependent compute phases into one command sequence can improve
  model throughput dramatically.
- Cache visibility must be explicit at CPU/GPU ownership transitions.
- A timeout, incorrect result, GPU fault, process failure, and kernel panic are
  different outcomes and must not be collapsed into one label.

The current OpenGL backend waits for completion before returning from its
native submission boundary. This is simple and correct for current tests, but
it is not a proof of asynchronous multi-context scheduling.

## Graphics path

Mesa supplies API objects, validation, shader-language handling, version
derivation, and state normalization. The Gallium backend translates normalized
state into native resources and commands.

The presentation path keeps rendering and scanout resources explicit. Direct
GPU rendering into alternating presentation buffers is proven. Offscreen
resources can instead be read back into deterministic test receipts.

Video decode remains a separate subsystem. Decoded caller-owned surfaces can
enter this graphics path as sampled textures, but shader execution does not
perform the codec decode itself.

## Compute path

The compute runtime binds storage buffers and constants, dispatches ordered
kernels, and validates the result against CPU references. Transformer
inference adds:

- packed model storage;
- model-specific tensor descriptors;
- RMS normalization, matrix projections, attention, activation, and residual
  kernels;
- resident per-layer key/value caches;
- prompt-prefill batching;
- one ordered GPU graph per generated token; and
- CPU tokenization, selection, and UI integration.

The runtime is model-profile driven. It is not a general graph compiler, and a
new architecture or tensor geometry can require new scheduling and kernels.

## Design rules established by the experiments

1. Derive advertised capabilities from implemented behavior; never override a
   version string to reach a target.
2. Validate descriptors and shader metadata before submission.
3. Keep GPU mappings stable for the full descriptor lifetime.
4. Match compiler register and wave assumptions to dispatch state.
5. Prefer one ordered submission for tightly dependent phases.
6. Separate logical API dimensions from physical storage geometry.
7. Use exact CPU, pixel, token, or hash oracles instead of appearance alone.
8. Preserve failed results and replace bytes rather than silently reclassifying
   them.
9. State firmware, workload, and measurement boundaries with every claim.
10. Keep proprietary material outside source control and publications.

## Not established

- A general-purpose Vulkan, ROCm, CUDA, or OpenCL stack.
- Safe concurrent access from multiple native processes.
- Fully asynchronous OpenGL fences or shared-context behavior.
- Universal model-architecture support.
- Peak theoretical GPU throughput.
- Cross-firmware equivalence beyond tested cases.
- Formal OpenGL conformance.
