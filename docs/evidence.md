# Evidence and limits

## Evidence method

Controlled GPU and CTS findings use a bounded native application and a declared
oracle. Owner-reported interaction and sampled gameplay observations are labeled
separately; they do not replace exact numerical checks. The evidence chain records:

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
| Host-checked | Compilation, linking, host tests or archive verification passed; no console execution is implied |
| Owner-observed | Physical behavior reported by the owner; not a substitute for a complete numerical oracle or frame-time trace |
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
| Historical frozen Core 3.3 project campaign | Controlled | 37,404 Pass plus 2,140 reviewed NotSupported across four configurations; six disclosed CTS adaptations |
| Khronos certification | Not claimed | Project acceptance is not certification |
| Later optimized SDK compatibility | Controlled sample | G7 SDK passes 204 selected executions; later binaries require their own scoped evidence |
| Small-scene graphics throughput | Controlled | 30-second windowed runs at ~119.88 FPS at 1080p–4K; not a general game or perfect-pacing result |
| Native 1440p120 and 4K120 output | Controlled plus console HDMI evidence | September 8 unchanged apps, exact pixels and matching 119.88 Hz negotiation; independent per-run TV refresh observations remain separate |
| SDL fixed high-resolution profiles | Controlled functional checks | 180 frames/two exact pixels per resolution, matching HDMI, normal-output restoration and clean teardown; not measured SDL FPS |
| 128-cube graphics throughput | Controlled | ~59.94 FPS ordinary and instanced at 1080p; small texture working set |
| Offscreen-copy improvement | Controlled plus host-checked | Matched 1080p case 14.10 to 19.98 FPS, with pixels/completion/teardown and scalar-equivalence checks |
| Bounded graphics stability | Controlled plus instrumented normal session | Ten-minute session and five native launch/exit cycles; CPU-owned heap only |
| Yamagi 4K gameplay | Owner-observed | Later optimized development candidate reported at 120 FPS throughout tested areas; bounded scenes and texture-quality limits |
| Yamagi resolution switching | Controlled timing and lifecycle | Six switches across 1080p/1440p/2160p; steady 120-frame windows measured separately from restart time; no new exact pixel oracle |
| Yamagi CI game download | Host-checked plus controlled native startup | All 77 files verified; exact CI executable's 4K menu median 119.880240 FPS over 12 windows; clean close/release; no separate manual CI gameplay run |
| Fresh SDK distribution build | Host-checked | Clean CI compilation, 344 exports, three relocated links and archive checksums; no console qualification |
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
- a completed frozen four-configuration campaign with individually reviewed exclusions;
- later optimized-candidate samples kept separate from that baseline;
- installed-SDK ImGui, NanoVG and Sokol checks;
- preserved functional failures that led to general fixes; and
- clean bounded lifecycle receipts.

The historical full campaign is complete within its declared scope. Its
39,544 accounted results are not 39,544 passes and do not qualify every later
compiler, runtime or application. The remaining work includes broader workload
performance, resource/recovery stress and validation of subsequent changes.

### Graphics evidence identities

The historical September 8 record summarizes the separately controlled `ps5-opengl` source
snapshot `bd1c77fdd1cbef65092525444dabc53196ea9746` and the reviewed `ps5-yamagi`
handoff at `b711c4f90c4e5ec183af5aa31db8589586f0d69a`. References below are project
and document identifiers, not private repository links or personal paths. Raw
receipts and full artifact inventories remain with their respective projects.

| Record | Identity and source document | Scope |
| --- | --- | --- |
| Historical full campaign | Runtime `0a15d8fa82f3f96cf071be927ad11422964a16ba`; `ps5-opengl` document `docs/validation.md` and export `validation/2026-09-07/` | Four complete configurations; 37,404 Pass and 2,140 reviewed NotSupported; 15 CTS cycles |
| Graphics performance progression | G5d `610e6a3`, G6 `ec9ac2c`, G7 `cef6c1b`; `ps5-opengl` document `docs/performance.md` | Separately frozen high-refresh, close-path and 3D candidates; not one combined full-matrix result |
| Optimized sampled SDK | G7 runtime `cef6c1b`; `ps5-opengl` document `docs/sdk-bundle.md` | 204/204 selected executions and independent consumer checks; preserves its own bytes |
| Offscreen and sustained session | G9 `891dab7`, G10 instrumentation `725f6eb`; `ps5-opengl` document `docs/offscreen-stability.md` | Matched offscreen comparison, 600-second session, five launches/15 EGL sessions |
| Initial game-derived findings | Reviewed `ps5-yamagi` document `docs/performance-handoff.md`; then-private G7 derivative | Allocation policy, scoped texture maintenance and bounded 1080p gameplay observations; not merged SDK acceptance |
| Clean CI-built SDK | Source snapshot `bd1c77f`; `ps5-opengl` document `docs/ci-releases.md` and CI run `34183840335` | Build/link/package verification only; these binaries were not run on the console |

Later display follow-up: `ps5-opengl` source companion `b28f96c` records G37–G40
in `docs/high-resolution-120-plan.md`. The frozen ImGui build source is
`3cdc90b`; SDL profile integration is `ac2a52a`. Subsequent documentation commits
are not rebuilds. The implementation's `docs/performance.md` publishes the exact
SDK/runtime/executable SHA-256 identities. The same 4K executable's HDMI1/HDMI4
comparison establishes a connection-path limitation for that setup; the native
1440p result follows a separately owner-selected resolution. It does not qualify
every TV input, cable, capture device, new SDK build or HDMI configuration.

The owner confirmed 4K120 for a preceding stream on HDMI4 and 1440p resolution
before its OpenGL test. Neither statement is recorded as independent TV refresh
verification of every later OpenGL/SDL run. The four bounded applications have
their own console negotiation, pixels, restoration, teardown and health evidence;
none inherits the historical full CTS campaign or changes the compute results.

The original CTS revision is `cf7edb26d3be2d8763595ed08fdc41f3c1b1966f`.
Six disclosed changes cover platform build/package routing, portable I/O and a
negative compute-shader version guard, with the platform overlay supplied
separately. The exhaustive swizzle and LOD-bias workloads remain unchanged. Do
not describe this runner as untouched upstream CTS or imply formal certification.

The game case and all graphics hardware records above are scoped to the recorded
firmware-6.02 console. Reusing a concept across graphics and compute does not
transfer either project's correctness or performance acceptance to the other.

### Yamagi 120 Hz release identities

The September 11–12 follow-up is published in
[Yamagi Quake II v0.2.0-alpha.1](https://github.com/blackbearreloaded/ps5-yamagi-quake2/releases/tag/v0.2.0-alpha.1).
Its [validation receipt](https://github.com/blackbearreloaded/ps5-yamagi-quake2/releases/download/v0.2.0-alpha.1/validation.json)
separates the automated development run from the exact CI-download startup check.
The [release sources and build description](https://github.com/blackbearreloaded/ps5-yamagi-quake2/blob/65b2a33333cd23d561ffea84750cfbe55783a619/docs/building.md)
identify the compiler, matching shader-compiler headers, runtime configuration
and included patch series. No implementation binaries or raw captures are copied here.

| Record | Identity |
| --- | --- |
| Published game source and CI checkout | `65b2a33333cd23d561ffea84750cfbe55783a619` |
| Public OpenGL base, before game-specific runtime patches | `32ca4d4e16c0f29d75b4ae82b74c2df6e1e067bf` |
| Frozen game SDK manifest SHA-256 | `e73d2a40c5c67bd15c6cf1805c2ec4890286e311811d0546f0656a36e267fa99` |
| Frozen game runtime archive SHA-256 | `f4641d706911fc5b72aa780efca52744bedf9173c6592a17815417abc90eeccf` |
| CI game archive SHA-256 | `4fa5da07498fe64ffc67229eed2347e0ddffd5073f1fea248ba5e69307229a70` |
| CI executable SHA-256 | `6979f94073122a1170b3eab8208a6cdbae5f0be511b9d4cb6efd0bc4083dcd9f` |
| Automated mode-cycle development source | `99678e05aad3fbd79f97b1afe8f9ea9e1549ff5e` |
| Automated mode-cycle executable SHA-256 | `3e6554d7942849de9fc49e765c5a36bfdcbd7bd430337bf7d36cde6ee714d457` |

[Actions run 34666849359](https://github.com/blackbearreloaded/ps5-yamagi-quake2/actions/runs/34666849359)
built the game independently using the frozen prebuilt SDK; it was not an
independent runtime-library rebuild. The downloaded game's native startup test
therefore qualifies those exact game bytes for that bounded scenario. It does
not transfer the development candidate's manual gameplay observations to the
CI executable, or qualify a rebuilt library merely because its source matches.

The [benchmark record](benchmarks.md#public-120-hz-follow-up) reports each
observation's workload and timing boundary. Six successful resolution changes
do not establish recovery under device loss, memory exhaustion or suspend/resume.
The current game SDK has focused checks and bounded evidence on one firmware-6.02
console; it does not inherit the historical full CTS campaign. The proposed
benchmark matrix remains pending and adds no completed hardware cases.

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

### Source builds versus hardware-qualified binaries

The September 8 CI run independently built the graphics SDK from the pinned public
dependencies on a clean GitHub-hosted runner. Host regressions, compiler checks,
all 344 Core exports, relocated Make/pkg-config/CMake consumer links and archive
checksums passed. The distribution includes sources, licenses and provenance.
This establishes a working source-to-SDK build path, not GPU execution, identical
bytes across toolchain updates or reproduction of an older console campaign.

Keep three identities distinct: source snapshot, compiled artifact and hardware
receipt. A newly compiled SDK does not inherit older acceptance because its source
is related. Likewise, the five-minute historical baseline, later high-refresh
microbenchmarks and ten-minute normal session are not one unchanged executable.

For the fresh CI archive, the recorded SDK manifest SHA-256 is
`25b3d3112630f3f45248041f3530c77b746309f2310444d96c9276e4bd06e0a9`.
It differs from the sampled hardware-tested SDK and is explicitly host-checked,
not console-validated. No binaries or raw CI/console logs are copied here.

## Pending work

1. Validate changed graphics paths with affected exact-oracle tests and frozen
   candidate identities; retain the historical complete matrix as a separate record.
2. Reduce offscreen layout-copy costs and test heavier scenes without assuming
   that windowed presentation throughput transfers to sampleable targets.
3. Review reusable game-runtime changes in PS5 OpenGL with affected correctness
   tests and the [proposed game-derived benchmark matrix](benchmarks.md#proposed-game-derived-benchmark-matrix).
   Compare individual changes before combined acceptance; extend normal sessions,
   GPU-memory accounting and recovery testing separately.
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
