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

## OpenGL observations

OpenGL development has prioritized correctness rather than throughput. One
controlled Core 3.3 animation alternated two presentation buffers and observed
approximately 4-7 ms GPU retirement for its exact workload. This is sufficient
to prove direct presentation and changing GPU output, but not to characterize
driver or game performance.

No aggregate OpenGL benchmark is published until the immutable CTS campaign
and representative consumer-application tests complete.

## Interpretation limits

- Do not compare cached and streaming matrix rows as if they measure the same
  memory behavior.
- Do not convert end-to-end token rates into raw GPU arithmetic throughput.
- Do not compare model sizes without matching architecture, context, output
  length, quantization, and cache state.
- Do not treat one-frame OpenGL timing as a sustained frame-rate result.
- Do not infer another firmware's performance.
