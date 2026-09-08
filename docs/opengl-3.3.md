# OpenGL 3.3 findings

## Current status

The implementation creates a real Mesa-derived OpenGL 3.3 Core context and
reports GLSL 3.30. It does not force either version through an override. Mesa's
ordinary cumulative feature predicates determine the result from the Gallium
screen capabilities and supported formats.

All 344 OpenGL 3.3 Core commands are present in the public link surface. The
historical frozen four-configuration campaign completed on September 7, 2026,
with 37,404 Pass and 2,140 individually reviewed NotSupported results. This is
project acceptance, not Khronos certification. Later optimized runtimes have
separate focused evidence and do not inherit the full campaign.

OpenGL provides a standards-facing validation workload for the broader GPU
findings: shader translation, resource layout, synchronization, presentation and
performance. It is not required for the independent compute frontend.

## Stack

```text
Application
  -> public EGL and OpenGL headers
  -> Mesa OpenGL state tracker
  -> Gallium pipe interface
  -> PS5 resource/state/command backend
  -> target shader compiler
  -> native graphics submission
  -> offscreen oracle or VideoOut presentation
```

The public consumer SDK provides regular static archives and Make,
`pkg-config`, and CMake integration. Application code uses standard EGL,
OpenGL, and KHR headers and does not include native GPU structures.

## Context and shader language

Hardware-proven behavior includes:

- native fullscreen EGL lifecycle;
- explicit OpenGL 3.3 Core profile creation;
- Mesa-derived version, extension, format, and limit reporting;
- runtime GLSL 3.30 vertex and fragment compilation;
- geometry-shader compilation, linkage, varyings, textures, and uniform
  buffers;
- explicit vertex and fragment-output locations;
- integer and bit-preserving shader operations;
- ordinary uniforms, uniform blocks, and reflection; and
- large shader programs using dynamically sized staging storage.

Invalid requests are rejected rather than silently downgraded to another
profile.

## Draw and vertex input

| Family | Proven scope |
| --- | --- |
| Primitive assembly | Points, lines, line strips/loops, triangles, strips, and fans |
| Indexed draws | Unsigned 8-, 16-, and 32-bit indices in tested ranges |
| Base vertex | Zero, positive, and negative base values with bounded effective indices |
| Primitive restart | Mesa lowering with restart-separated connected primitives |
| Instancing | Instance count, instance ID, base instance metadata, and divisor-one attributes |
| Vertex arrays | Multiple layouts, offsets, strides, and retained object state |
| Vertex formats | Float, half-float, normalized integer, packed 2_10_10_10, BGRA8, signed integer, and unsigned integer paths |
| Raster modes | Fill, width-one line, fixed-size point, culling, provoking vertex, and front-facing semantics |

Software lowering is used where it is safer and already provided by Mesa. A
software-expanded path is not advertised as a native hardware feature.

## Textures and samplers

| Area | Proven scope |
| --- | --- |
| Targets | 1D, 1D array, 2D, 2D array, 3D, rectangle, cube, and multisample |
| Dimensions | Arbitrary non-power-of-two dimensions within advertised limits |
| Mipmaps | Upload, generated chains, explicit LOD, min/max clamp, and level selection |
| Filtering | Nearest and linear filtering, including mip filtering and seamless cube edges |
| Addressing | Repeat, mirrored repeat, edge clamp, and border clamp |
| Sampler state | Separate sampler objects, arbitrary border color, compare mode, and component swizzle |
| Normalized formats | One- to four-channel 8/16-bit normalized and signed-normalized families |
| Float formats | 16/32-bit float, shared-exponent RGB9_E5, and packed R11G11B10 |
| Integer formats | Signed and unsigned 8/16/32-bit representative render/sample paths |
| Packed formats | RGB10_A2 and integer variant |
| sRGB | Sampling and destination conversion control |
| Compression | RGTC1/RGTC2 through a validated CPU decompression fallback |
| Depth/stencil | D32F and combined depth/stencil storage, transfer, raw sampling, shadow compare, and 4x sampling |

An important layout finding is that logical mip dimensions and physical linear
storage extents are not always the same for non-power-of-two resources. Using
the target's ceiling-based physical packing fixed exact R8 and combined
depth/stencil repeat-mode cases at 11x131 while leaving OpenGL-visible
dimensions unchanged.

## Framebuffer and raster operations

Hardware-proven slices include:

- framebuffer completeness and bounded readback;
- color, depth, and stencil attachments;
- mixed attachment dimensions with validated clipping;
- four active color targets and an eight-target advertised limit;
- indexed color masks and independent blend state;
- source-alpha and dual-source blending;
- viewport, scissor, face culling, front-face orientation, and depth clamp;
- polygon offset for positive and negative slope or constant terms;
- sample coverage and sample masks;
- 4x multisample color and depth/stencil rasterization;
- per-sample fetch and resolve;
- same-format copy and scaled nearest/linear blits; and
- render-to-texture, layered rendering, and array-layer selection.

Selected blit, resolve, and compressed-format operations currently use CPU
fallback. They are functional paths, not performance claims.

## Queries, synchronization, and transform feedback

The implementation has bounded hardware evidence for:

- occlusion counters and predicate-based conditional rendering;
- transform feedback in interleaved and separate modes;
- append, instancing, primitive counts, and capacity bounds;
- serialized completion and public sync-object lifecycle; and
- monotonic timer-query behavior.

Timer queries currently use monotonic host timing. Conditional rendering is
synchronously evaluated. These satisfy the tested API behavior but should not
be presented as native asynchronous performance features.

## Presentation

The native context can render directly into alternating presentation buffers.
A controlled OpenGL animation produced changing GPU frame hashes, alternated
the two scanout slots, retired GPU work in approximately 4-7 ms in that exact
workload, and exited cleanly.

That early observation was a correctness result, not a sustained benchmark.
Later controlled candidates reach ~119.88 FPS in the small windowed ImGui scene
at 1080p, 1440p and 4K, and ~59.94 FPS in a 128-cube 1080p scene. Sampleable
offscreen targets remain substantially slower because of CPU layout copies.
See [graphics benchmarks](benchmarks.md#graphics-benchmarks) for timing, workload,
output-status and frame-pacing boundaries.

The later full-port shutdown path drains owned work, restores output state and
closes the presentation handle before releasing resources. It avoids a redundant
unregister operation; this is not proof of unregister-while-open or device-loss
recovery. Ten-minute session and repeated-launch evidence is separately scoped
in the benchmark document.

## Khronos test status

The native test application embeds the pinned `KHR-GL33` package and writes
machine-readable status plus a full QPA log. The official list has:

- 9,886 cases;
- four surface/seed configurations; and
- 39,544 total ordered executions for a complete campaign.

Targeted tests have already exposed and validated fixes for:

- large uniform-block shader staging;
- non-power-of-two physical mip packing;
- combined depth/stencil mip staging and sampling;
- texture addressing and repeat behavior;
- exact LOD-bias limits aligned with the open-source AMD driver policy;
- sample positions and multisample raster state;
- depth/stencil copy and blit behavior; and
- framebuffer and shader-language edge cases.

### Completed historical baseline

The frozen campaign accounts for every listed case in all four configurations:

| Result | Per configuration | Total |
| --- | ---: | ---: |
| Pass | 9,351 | 37,404 |
| Individually reviewed NotSupported | 535 | 2,140 |
| Accounted | 9,886 | 39,544 |

No gaps, duplicates, required-case failures, warnings, waivers or incomplete
results remain in the accepted set. The reviewed exclusions concern optional,
newer or upstream-defined inapplicable combinations, not accepted omissions of
mandatory Core 3.3 behavior. All 15 recorded CTS lifecycle cycles completed
uneventfully. Final installed-SDK ImGui, NanoVG and Sokol checks also passed;
they supplement the matrix rather than adding CTS cases.

The runner uses upstream CTS revision
`cf7edb26d3be2d8763595ed08fdc41f3c1b1966f` with **six disclosed adaptations** for
build/package routing, portable I/O and a negative compute-shader version guard,
plus the platform overlay. It is not an untouched upstream executable. Full
swizzle and LOD-bias workloads were retained. Frozen source and artifact identity,
ordered case accounting, exclusion review and clean lifecycle remain essential.

### Later candidates

The optimized G7 SDK passed 51 selected cases in each configuration: **204/204
Pass**, with no exclusions, warnings or waivers. That focused campaign and its
consumer checks do not replace the historical full matrix. Later G9/G10 changes
have their own performance and lifecycle checks, not inherited full-CTS or G7
binary acceptance. Game-derived runtimes and fresh CI binaries are separate again.
See [evidence identities](evidence.md#graphics-evidence-identities).

## Current limits

- Khronos certification is not claimed; later binaries have not rerun the full
  historical matrix.
- The compatibility profile and fixed-function OpenGL are not targets.
- Eligible draws and clears use bounded deferred batches with checked native
  completion; concurrent queues and shared-context behavior are not broadly proven.
- Some transfer, decompression, and resolve paths prioritize correctness over
  performance.
- Timer behavior is emulated with monotonic host time.
- Focused performance gains do not remove CPU fallbacks or establish arbitrary
  application performance.
- The present evidence is primarily firmware-specific.

## Claim boundary

The result establishes the feasibility of a reusable, standards-facing OpenGL
3.3 Core stack on PS5 hardware through Mesa and a dedicated backend. It does
not establish formal Khronos conformance, peak performance, support for later
OpenGL versions, compatibility-profile behavior, or every firmware revision.
