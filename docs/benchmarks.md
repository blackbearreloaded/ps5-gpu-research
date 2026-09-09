# Controlled benchmarks

## Measurement policy

These are single-console controlled results, not confidence intervals or peak
hardware specifications. Every comparison must preserve:

- firmware and native application environment;
- exact shader/model and tensor dimensions;
- quantization layout;
- command-submission structure;
- cache and residency state;
- timing start/end boundaries; and
- correctness oracle.

For graphics, also preserve scene content, render-target usage, resolution,
sample count, texture quality, warm-up, pacing policy and runtime identity.
Distinguish render dimensions from reported output dimensions and completed-frame
throughput from output refresh rate or isolated GPU execution time.

Short shader timings can be dominated by queue startup and polling resolution.
Values labeled lower bounds include those costs and should not be compared to
vendor peak specifications.

## Compute microbenchmarks

| Workload | Result | Interpretation |
| --- | ---: | --- |
| 4 MiB storage-buffer fill, 16,384 groups | 13.233 ms; 302.274 MiB/s | Correctness scale test dominated by 1 ms polling |
| 4,096x4,096 packed projection, strided baseline | 12.042 ms; 1,328.682 MiB/s | One-lane row mapping |
| Same dimensions, cooperative contiguous workgroups | 1.149 ms; 13,925.152 MiB/s | 10.48x controlled improvement |
| 16,384x4,096 cooperative packed projection | 9.982 ms; 6,411.540 MiB/s | Larger working set, exact output |
| Same projection with parallel reduction | 7.657 ms; 8,358.364 MiB/s | 30.36% improvement over serialized reduction |
| 16,384x4,096 packed Q4-by-Q8 projection | 3.276 ms; 20.485 GMAC/s | 32 MiB packed workload, exact rows |
| GGML Q4_0 with FP16 scales converted in shader | 8.731 ms; 7.686 GMAC/s | Scale conversion was expensive |
| Q4_0 with scales expanded to FP32 at load time | 1.131 ms; 59.336 GMAC/s | 40 MiB workload, exact rows; best controlled projection lower bound |
| One real 576x576 Q4_0 model tensor | 4.362 ms | Correctness result below useful timing size |
| 256 cached executions of that tensor in one command sequence | 3.327 ms aggregate | Demonstrates persistent command construction; cache-assisted |

The matrix rates are not full-model rates. Attention, normalization, activation,
cache traffic, logits, command construction, and CPU coordination all affect
end-to-end inference.

## Model benchmarks

| Model/profile | Workload | Result |
| --- | --- | ---: |
| SmolLM2 135M | 64-token controlled short-context path | Approximately 60 tokens/s in the optimized single-submit graph |
| SmolLM2 135M | Native factual chat | 736 ms total for the controlled prompt/answer |
| SmolLM2 360M | Five-token prompt plus 12 exact tokens | 0.168 s prefill; 0.685 s total |
| SmolLM2 360M | Near-limit 501-token prompt plus six tokens | 33.435 s prefill; 33.934 s total |
| SmolLM2 1.7B | Short reference prompt and continuation | Approximately 20 decode tokens/s |
| SmolLM2 1.7B | 260-token prompt plus 64-token budget | 1.846 s prefill; 4.999 s total |
| SmolLM3 3B | Short exact-token run | 0.114 s prefill/first logits; 0.515 s total |
| SmolLM3 3B | 449-token prompt plus 64-token budget | 6.023 s prefill; 11.844 s total; about 10.8 decode tokens/s |
| Mistral 7B Q4_0 | 64-token sustained run | 0.083 s prefill/first logits; 1.919 s total; 34.3 decode tokens/s |
| Mistral 7B Q4_0 | 1,888-token multi-turn prompt | 7.698 s prefill; 12.953 s total |
| Mistral 7B Q4_0 | 4,095-token boundary prompt | 84.291 s cold prefill; completed through position 4,095 |
| Qwen3.5 9B quantized | End-to-end native chat and model switch | Coherent output and expected-token switch-back; no general speed claim yet |

## Loading and memory

| Profile | Packed image or mapped footprint | Result |
| --- | ---: | --- |
| Mistral 7B | 4.66 GB packed image | 1.455 s concurrent load plus 0.079 s visibility step |
| Mistral 7B | Approximately 5.42 GiB mapped | Weights, work/KV, compact globals, and alignment |
| SmolLM3 3B | Approximately 2.02 GiB mapped | Controlled native runtime |
| SmolLM2 1.7B | Approximately 2.47 GB mapped | Model, resident decoder cache, and work/KV |
| Qwen3.5 9B | Approximately 7.4 GiB mapped | Current hybrid recurrent/attention runtime |

The 7B final-order loader is roughly 34x faster than the original per-tensor
loading route in the same investigation. Later turns reuse resident data and
report zero model reload time.

## Graphics benchmarks

The September 7 results and separately labeled September 8 follow-up use OpenGL
as the workload frontend on one firmware-6.02
console. They are controlled performance findings for later optimized candidates,
not a new full CTS campaign. Source identities and report names are in
[the evidence record](evidence.md#graphics-evidence-identities).

### Small windowed scene

The same ImGui scene ran for 30 measured seconds after 30 warm-up frames at each
render size. Each run completed 3,597 measured frames, passed both warm-up pixel
probes, and retained checked draw completion, presentation and teardown.

| Render size | Completed FPS | p95 / p99 frame interval (ms) |
| --- | ---: | ---: |
| 1920x1080 | 119.883028 | 9.262385 / 9.338802 |
| 2560x1440 | 119.881864 | 8.874127 / 8.985345 |
| 3840x2160 | 119.882543 | 9.014852 / 9.283553 |

The matched 4K diagnostic baseline measured 59.942610 FPS. Reusing presentation
cache maintenance within a batch raised it to 70.089393 FPS; avoiding unnecessary
depth/stencil maintenance while those tests were disabled reached 119.882543 FPS.
The scene and GPU commands were unchanged, and completion checks were not weakened.
Both output status APIs reported 119.88 Hz and 3840x2160 output extents, then
restoration to 59.94 Hz. This is not independent HDMI timing or a fresh physical
TV/controller observation. Average 120-class throughput is not perfect pacing,
and the later candidate has not requalified the earlier matrix's 4K90 case.

### Native HDMI qualification (September 8 follow-up)

Later frozen apps on the same firmware-6.02 console distinguish completed
rendering from negotiated HDMI output. The tested display was a Hisense 55U78N
using HDMI4. Both 30-second ImGui workloads completed 3,597 measured frames:

| Render dimensions | Measured seconds | Completed FPS | Console-reported active HDMI |
| --- | ---: | ---: | --- |
| 2560x1440 | 30.004589 | 119.881660 | 2560x1440 at 119.88 Hz |
| 3840x2160 | 30.004094 | 119.883639 | 3840x2160 at 119.88 Hz |

Both pixel oracles passed. A separate standard-SDL consumer completed 180
frames and two exact pixel checks at each size with matching HDMI timing.
All four native cycles restored 59.94 Hz at their selected resolution, closed
cleanly and passed environment-health checks. The SDL runs are functional
checks, not SDL FPS measurements or additional long-session evidence.

The useful controlled comparison is physical: the unchanged 4K executable
negotiated 1080p120 on HDMI1, then 4K120 on HDMI4. No renderer rebuild or app
metadata change was needed. The display's
[official quick-start guide](https://assets.hisense-canada.com/assets/ProductDownloads/486/f1ae562e0c/QSG-English-55-65-75U78N.pdf)
(printed page 5) identifies HDMI1/2 as 4K60 and HDMI3/4 as 4K144 inputs.
Native 1440p additionally followed the owner's console output selection;
1440p rendering alone did not establish a native 1440p signal.

Three observations must remain distinct: framebuffer dimensions, completed
application FPS, and the negotiated link/independent sink signal. A render-size
HUD plus a TV game bar showing only 120 FPS is not proof of 4K input. Correlate
resolution and refresh during the same workload; a menu or stale display banner
can describe a different state. These findings concern presentation, not video
decoding correctness or general compute throughput. They do not establish
arbitrary-game FPS, perfect pacing, HDR pixel accuracy or another firmware.
See the [OpenGL qualification record](https://github.com/blackbearreloaded/ps5-opengl/blob/b28f96c/docs/high-resolution-120-plan.md)
for the separately identified runs; raw logs and executables are not copied here.

### Textured 3D and submission cost

The 128-cube workload uses two small textures at 1080p, with 30 measured seconds
per draw mode. Its final candidate reaches 59.941451 FPS for ordinary draws and
59.940062 FPS for instanced draws, each completing 1,799 measured frames. All
2,052 pixel probes across the two runs pass, together with exact completion
accounting and clean teardown. Ordinary-frame p99 is 17.62 ms.

| Controlled change | Ordinary-draw FPS before / after | Interpretation |
| --- | ---: | --- |
| Routine per-draw success traces made opt-in | 4.57 / 11.99 | Unbuffered diagnostic I/O was a substantial CPU cost; error and lifecycle checks remained |
| Batch-local depth maintenance reuse at the 128-entry stage | 14.98 / 29.97 | The same idea had negligible benefit with smaller batches; bottlenecks interact |
| Batch capacity 128 to 256 | 29.97 / 59.94 | The scene's clear and 128 draws can retire together without weakening hazard checks |

These are successive matched comparisons, not interchangeable binaries. Earlier
mixed-clear ordering separately improved the instanced case from 29.97 to 59.94
FPS. The result does not characterize heavy shaders, large texture working sets
or sustained performance at larger object counts.

### Sampleable offscreen targets

The same 1080p ImGui offscreen workload, with two retired draws per frame, remains
limited by CPU layout copies. A matched 30-second comparison gives:

| Runtime | Completed FPS | Frames / elapsed seconds | Mean / p95 / p99 render time (ms) |
| --- | ---: | ---: | ---: |
| Published-SDK control | 14.095217 | 423 / 30.010180 | 70.942 / 83.484 / 85.288 |
| Contiguous-copy candidate | 19.981406 | 600 / 30.027917 | 50.042 / 51.066 / 51.472 |

The gain is approximately 41.8%, with pixel probes, completion and teardown
passing. Host sanitizer checks compare the actual copy helper against scalar
results across formats, tails, mip levels and layers. This optimizes CPU work;
it neither removes the transfers nor achieves 60 FPS. Do not compare it to the
windowed row as if render-target storage and timing boundaries were identical,
or interpret the fallback's rate as the GPU's native render-to-texture ceiling.

### Sustained session and lifecycle

A later 1080p normal TV-demo session completed 35,942 frames and 21 pixel checks
over 600 seconds, averaging about 59.90 FPS. Thirty-second windows ranged from
59.87 to 59.93 FPS. It recorded 112 widget changes, so it was not an unchanged-input
performance comparison.

Owned heap use increased by 448 bytes/two blocks by 90 seconds, then stayed flat
through the last steady sample at 570 seconds. Five separate native launch/exit
cycles completed 15 EGL sessions and 90 checked frames; post-session owned heap
was 9,375 bytes/21 blocks every time, without allocation failures or ambiguous
zero-size reallocation accounting. All cycles closed cleanly.

This covers the instrumented CPU allocator, not GPU direct mappings, foreign
allocators or process resident memory. It does not establish multi-hour stability,
absence of every leak, deliberate memory exhaustion, suspend/resume or device-loss
recovery. Earlier focused full-port teardown tests removed the redundant
buffer-unregister busy warning while preserving close and restoration checks.

### Real-application corroboration: Yamagi

The separately reviewed Yamagi port combines game renderer changes with a private
graphics-runtime derivative. The owner reported stable 60 FPS while moving and
firing. Its final capture contains 66 timing samples taken every 60 frames:
median 16.735 ms, sampled p95 33.499 ms. Heavier entity scenes still reached
approximately 33 ms; these are not all-frame statistics or a universal 60 FPS
guarantee.

The two reusable runtime findings were allocation policy by resource lifetime
and reuse of unchanged texture maintenance within a retained batch. The latter
reduced the world pass from approximately 10–12 ms to 2–3 ms in the compared
captures. These changes were not merged into the canonical SDK at the reviewed
snapshot. The game also uses material batching, streaming changes, particle quads
and single-level textures; the texture choice can cause distant shimmer. The
overall game result is not an isolated SDK comparison or a new conformance result.

## Interpretation limits

- Do not compare cached and streaming matrix rows as if they measure the same
  memory behavior.
- Do not convert end-to-end token rates into raw GPU arithmetic throughput.
- Do not compare model sizes without matching architecture, context, output
  length, quantization, and cache state.
- Do not treat one-frame OpenGL timing as a sustained frame-rate result.
- Do not equate average FPS with perfect pacing or compare unlike presentation
  and sampleable-offscreen workloads as a single GPU-speed metric.
- Do not combine the acceptance of different runtime binaries or infer that
  host-only CI checks execute graphics on a console.
- Do not infer another firmware's performance.
