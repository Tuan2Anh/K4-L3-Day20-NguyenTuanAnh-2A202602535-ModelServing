# Bonus - Context-length sweep (prefill cost)

Host `Linux-x86_64` · llama.cpp `b10488` ·
`threads=14` `ngl=0` · RAM 15.3 GB

| Prompt tokens | Prefill (tok/s) | TTFT contribution (ms) | vs linear scaling |
|:--|--:|--:|--:|
| 256 | 140.1 | 1827.7 | 1.00x |
| 1024 | 152.0 | 6737.7 | 0.92x |
| 2048 | 145.9 | 14032.2 | 0.96x |
| 4096 | 135.5 | 30219.9 | 1.03x |
| 8192 | 117.3 | 69844.0 | 1.19x |

At 8192 tokens, prefill costs **69844 ms** --
1.19x what linear scaling from the smallest point would predict. That excess
is attention's O(N^2) term becoming visible, and every millisecond of it lands in TTFT
before the user sees a single token.

Either way, this is the number to remember when someone proposes stuffing more retrieved
context into a RAG prompt "because the context window allows it". Prefill is paid in full,
on every request, before the first token appears.

## Your finding

Prefill latency grows dramatically from **1.83s at 256 tokens** to **69.84s at 8192 tokens**.

Key takeaways:
1. **The Quadratic Bend ((N^2)$ Attention):** Below 2048 tokens, prefill scales roughly linearly (~145-152 tok/s). However, between 4096 and 8192 tokens, throughput degrades from 135.5 tok/s down to 117.3 tok/s, pushing latency to **1.19x higher than pure linear scaling**. At 8192 tokens, the user waits nearly 70 seconds before receiving the very first token (TTFT = 69.8s).
2. **Impact on RAG Architecture:** This empirical finding proves why 'stuffing all retrieved chunks' into the prompt is catastrophic for user experience. If each retrieved chunk is ~500 tokens, retrieving 10 chunks (~5000 tokens) causes TTFT to exceed 40 seconds on CPU. A production RAG pipeline on this hardware should strictly budget retrieved context to <= 1000-1500 tokens (2-3 chunks) and rely on reranking rather than brute-force context stuffing.
3. **Disaggregated Serving Rationale:** Because prefill at 8k tokens takes 70 seconds of pure compute, running prefill on the same device as decode would starve all concurrent decode streams for over a minute, perfectly demonstrating the need for disaggregated prefill/decode serving.
