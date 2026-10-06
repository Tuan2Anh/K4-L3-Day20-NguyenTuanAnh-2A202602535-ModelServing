# 01 - Tune: thread-count sweep

Model `Qwen3.5-0.8B-Q4_K_M.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **14 physical · 20 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 13.9 | 47% |
| 7 | 29.7 | 100% |
| 14 | 26.3 | 89% |
| 20 | 20.1 | 68% |
| 40 | 12.2 | 41% |

**Best**: `-t 7` at 29.7 tok/s
**Slowest tested**: `-t 40` at 12.2 tok/s (2.44x spread)
**Against the physical-core default** (`-t 14`, 26.3 tok/s): 1.13x

Use this in your run:

```bash
LAB_N_THREADS=7 make bench
```

## Your explanation

The knee of the curve sits sharply at **-t 7** (29.7 tok/s), peaking well before the total physical core count (14 cores, 26.3 tok/s) and dropping drastically as thread count increases to 20 (20.1 tok/s) and 40 (12.2 tok/s).

This behavior is explained by the hybrid architecture of the Intel Core i7-12700H, which consists of **6 high-frequency Performance cores (P-cores)** and **8 low-power Efficient cores (E-cores)**:
1. **P-core saturation and straggler effect (-t 7 vs -t 14):** At `-t 7`, computation is handled almost entirely by the fast P-cores with dedicated high clock speeds and ample L3 cache. When jumping to `-t 14` (all physical cores), the work is distributed across both P-cores and much slower E-cores. Because decode matrix multiplication requires thread synchronization across barriers, the slower E-cores become stragglers that force fast P-cores to wait, degrading performance by 11% (from 29.7 to 26.3 tok/s).
2. **Hyper-threading and cache thrashing (-t 20):** Hyper-threading logical cores share execution units and L1/L2 cache with their parent P-cores. During decode, which is strictly memory-bandwidth-bound, adding hyperthreads does not double compute; instead, it causes cache thrashing and worsens contention on the dual-channel memory bus, dropping throughput to 20.1 tok/s.
3. **Severe oversubscription (-t 40):** At 2x logical cores, excessive OS context-switching overhead and memory bus queueing drop throughput to 12.2 tok/s, which is even slower than a single thread (13.9 tok/s).

Therefore, setting `LAB_N_THREADS=7` delivers the optimal balance of P-core utilization and memory bandwidth without straggler or synchronization penalties.
