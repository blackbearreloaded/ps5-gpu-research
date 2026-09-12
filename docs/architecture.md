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

### Allocation policy follows resource lifetime

The Yamagi game investigation found that persistent texture allocations could
occupy the arena also used by frequent transient buffers. Buffer allocation then
fell back to more expensive checked direct allocations. Keeping textures on the
existing direct-allocation path preserved arena space for transient work without
increasing the pool size. This was allocator-policy pressure, not a demonstrated
exhaustion of the console's total GPU-visible memory.

The initial observation came from a game-specific graphics-runtime derivative.
The later public game SDK has its own [source and evidence identities](evidence.md#yamagi-120-hz-release-identities);
neither observation is a general allocation-failure qualification. Stable mappings,
bounds checks, fallback allocation and retirement ownership remain necessary.
The transferable lesson is to distinguish resource residency, lifetime and
allocation frequency before treating a larger pool as the solution.

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

The OpenGL backend can defer eligible API work into bounded batches while still
checking completion at its native submission boundary. Deferral is not proof of
asynchronous multi-context scheduling. Referenced resources remain retained until
their work retires; CPU access and incompatible operations drain pending work.

### Batching and cache maintenance

The graphics experiments extend the compute finding about submission granularity:
many small draws can spend more time in CPU preparation, logging, visibility
maintenance and completion waits than in useful GPU execution. Larger bounded
batches improved the same 128-cube scene, while repeated depth-cache maintenance
only became a meaningful throughput improvement after other bottlenecks changed.

The measured cache optimizations rely on ownership, not on disabling synchronization:

- repeated presentation-buffer maintenance can be reused within the same batch;
- unused depth/stencil storage need not receive the work required for active tests;
- required CPU-write/readback visibility and completion checks remain intact; and
- reuse ends when the batch drains or the relevant resource backing changes.

The game-derived texture optimization applies the same principle to eligible
read-only linear fragment textures and lightmaps, within retained, unsubmitted
work. It does not establish cross-frame cache reuse, tiled-texture reuse or a
generic compute optimization. Reuse in another runtime revision requires the
affected resource-update and lifetime regressions.

CPU-wall profiling includes waits and is not an isolated GPU timestamp. Removing
per-draw success logging also produced a measured gain; diagnostic overhead must
be separated from hardware throughput. See [controlled graphics benchmarks](benchmarks.md#graphics-benchmarks).

### Presentation overlap and distinct resource lifetimes

The later Yamagi runtime overlaps CPU preparation of the next frame with
pending presentation. It still confirms presentation completion before GPU
work reuses a display buffer, and before incompatible CPU access or teardown.
This is a scoped scheduling change, not evidence of arbitrary concurrent GPU
queues or shared contexts.

Draw resources and command storage have different last-use boundaries. The
game-specific path confirms every original draw's completion before releasing
its resources. The final command allocation can remain owned until a separate
terminal completion is observed; it cannot return to the command pool or be
freed before that confirmation. Failed or unknown completion retains ownership.
These are source-derived lifetime rules in the published runtime, corroborated
by bounded game execution and focused checks, not a complete memory-race proof.

The optimized game's observed 60 FPS plateaus disappeared after scheduling,
batching, cache and diagnostic changes accumulated. The final result does not
assign the entire gain to one patch. The transferable diagnostic is to measure
CPU preparation, completion waits and presentation separately before attributing
a quiet scene's low FPS to GPU capacity. A CPU wall-clock interval that includes
a wait is not an isolated GPU execution measurement.

## Graphics path

Mesa supplies API objects, validation, shader-language handling, version
derivation, and state normalization. The Gallium backend translates normalized
state into native resources and commands.

The presentation path keeps rendering and scanout resources explicit. Direct
GPU rendering into alternating presentation buffers is proven. A sampleable
offscreen target follows a different storage path: the tested implementation
performs CPU-side linear/tiled layout copies around rendering. These costs remain
even when the offscreen texture is consumed by another GPU draw rather than read
back only for a test oracle.

Contiguous CPU copies improved the matched offscreen workload, but did not remove
the full-surface transfers. The ~120 FPS windowed result therefore cannot be
generalized to render-to-texture. Conversely, a slow fallback does not show that
the GPU is physically incapable of a native path. A GPU-resident alternative
remains implementation and validation work, not an established performance result.

Video decode remains a separate subsystem. Decoded caller-owned surfaces can
enter this graphics path as sampled textures, but shader execution does not
perform the codec decode itself.

### Fast-path thresholds across render sizes

The Yamagi integration exposed a GPU depth-clear eligibility threshold above
four million pixels. A full 1440p depth clear fell below that threshold and
used the CPU path, while 4K used the GPU path. Extending the existing eligibility
rule to cover all three supported game sizes removed this particular fallback
from 1080p and 1440p. Clear masks, scissor, format, target and state checks still
apply; this does not make every clear eligible for the same path.

An earlier game change also avoided clearing both an offscreen target and the
default target when only the active target needed it. These findings motivate
tests at size thresholds and across target usage, rather than assuming that a
smaller image must follow the same implementation and run faster. They do not
establish fast sampleable-offscreen rendering in general.

### Full-port teardown and bounded stability

Presentation buffers may still be owned by scanout after rendering has retired.
The tested full-port shutdown drains rendering and pending presentation, restores
the output mode, closes the owned presentation handle, and only then releases
its resources. Removing a redundant buffer-unregister operation eliminated the
previous busy warning on this close path. It does not prove that buffers can be
unregistered while keeping the port open. On failed drain, restoration or close,
resources must not be prematurely released.

A ten-minute normal graphics session and five separate native launch/exit cycles
extend the bounded evidence. Owned CPU-heap accounting stabilized during the
session and returned to the same post-session level across recreation checks.
It excludes direct GPU mappings, foreign allocators and process resident memory;
flat samples do not prove the entire driver is leak-free. See
[stability measurements](benchmarks.md#sustained-session-and-lifecycle).

The game adds a separate context-recreation lesson: a statically linked renderer
can outlive its graphics context. Cached object names and binding state must be
invalidated after successful graphics teardown before rebuilding resources.
Six native resolution changes passed after this correction; that bounded result
does not establish suspend/resume or device-loss recovery.

Render-buffer size, negotiated HDMI format and measured application FPS remain
separate state. The game's 1080p/1440p/2160p selection changes rendering and can
leave a console-managed 4K/120 Hz signal unchanged. This complements the earlier
HDMI-port comparison: neither a render-size setting nor a TV refresh banner alone
establishes all three quantities.

These additions summarize the September graphics record and the separately
reviewed game handoff; their source boundaries are listed in
[evidence identities](evidence.md#graphics-evidence-identities).

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
