# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 4.00 of 4 slots (100%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5724 |

Highest sampled value was **4.00 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

The peak batch width reached exactly **4.00 of 4 slots (100% utilization)**, with continuous batching actively grouping requests across all available decode slots.

Comparing this with the effective concurrency of **21.3** reported in `02-server-results.md`:
- `n_busy_slots_per_decode = 4.00` reflects the **physical compute saturation limit** set by `--parallel 4`. The engine can execute at most 4 requests concurrently in a shared decode step.
- The effective concurrency of 21.3 (calculated via Little's Law: RPS x average latency) measures the total number of in-flight requests in the entire system, including requests waiting in the queue.
- This is corroborated by `requests_deferred = 46`: while the 4 compute slots were 100% occupied decoding in batches, 46 requests were held in the waiting queue.

Therefore, both numbers are consistent and trustworthy: 4.00 indicates max compute slot utilization, while 21.3 reflects system-wide queue pressure.
