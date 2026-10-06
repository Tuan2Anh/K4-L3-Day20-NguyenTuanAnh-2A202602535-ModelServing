# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=14` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 38 | 0.68 | 12000 | 21000 | 22000 | 8.0 | 0.0% |
| 50 | 43 | 0.74 | 32000 | 55000 | 57000 | 21.3 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.09x** (22% of linear) |
| P95 latency | **2.62x** |
| Effective concurrency at 50 users | 21.3 vs `--parallel 4` slots (occupancy/slot ratio 5.32) |

**Saturated.** Throughput delivered only 1.09x for 5x the offered load, and effective concurrency (21.3) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.09x while P95 moved 2.62x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

The server reaches full saturation well below 50 users (likely around 10-15 users). The definitive evidence:
1. **Throughput plateau:** Increasing offered load by 5x (from 10 to 50 users) only yielded a **1.09x** increase in RPS (from 0.68 to 0.74 req/s), showing throughput has hit a ceiling.
2. **Latency explosion:** P95 latency surged by **2.62x** (from 21.0s to 55.0s), and P50 surged from 12.0s to 32.0s.
3. **Effective concurrency vs slots:** Effective concurrency reached **21.3**, which is **5.32x** higher than the 4 available slots (`--parallel 4`), matching the peak metric of 46 deferred requests. The extra ~34 seconds of latency is pure **queue time**, not compute time.

If our target SLO is P95 <= 25s, then at 10 users we maintain nearly 100% goodput (P95 = 21s), but at 50 users goodput drops to near zero because almost all requests violate the SLO.

**What to change first to raise goodput@SLO:**
I would first change **`--parallel` from 4 to 8** (alongside setting `LAB_N_THREADS=7`). With 15.3 GB RAM, each slot for Qwen 0.8B (2048 ctx) consumes minimal memory, so doubling slots allows 8 requests to decode concurrently rather than queueing up, cutting queue delay significantly and expanding the goodput boundary.
