# 01 - Measure: latency baseline

Model `Qwen3.5 0.8B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=14` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `Q4_K_M` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| Q4_K_M | 0.50 | 2177 | 259 / 361 | 40.2 / 42.1 | 2743 / 3013 / 3013 | 24.9 |
| UD-Q2_K_XL | 0.39 | 2041 | 333 / 388 | 40.0 / 40.6 | 2853 / 2906 / 2906 | 25.0 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` and `Q4_K_M` decode within 2% of each other here, for 0.11 GB difference on disk.

## Your observation

On my machine (Intel i7-12700H, 15.3 GB RAM), the 2-bit quantization (`UD-Q2_K_XL`) reduces disk size by ~22% (0.39 GB vs 0.50 GB) and slightly reduces load time (2041 ms vs 2177 ms). However, decode throughput is virtually identical (25.0 tok/s vs 24.9 tok/s, TPOT 40.0 ms vs 40.2 ms), while prefill TTFT is actually 28% slower on 2-bit (333 ms vs 259 ms P50) due to higher dequantization computation overhead on CPU. Furthermore, 2-bit quantization degrades semantic coherence and response quality. Given that 0.50 GB easily fits in 15.3 GB RAM, trading off TTFT and response quality for just 110 MB of RAM is **not worth it**. `Q4_K_M` is the clearly superior choice for production serving.
