# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 8053.9 | 8054.1 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.1 | 5746.3 | 5746.5 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.1 | 8517.7 | 8517.9 |

Mean per stage (ms): embed **0.0** · retrieve **0.1** ·
llm **7439.3** · total **7439.5**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Based on the provided context, **Goodput** is more useful than raw throughput because it filters out requests that do not meet their specific Service Level Objectives (SLOs).

While raw throughput measures the total requests per second, Goodput specifically counts only those requests that satisfied the TTFT (Total Time to Failure) and TPOT (Total Time to Preload) targets. This filtering mechanism 

**What problem does PagedAttention actually solve?**

> PagedAttention solves the problem of **internal fragmentation in GPU memory** caused by storing key-value pairs (KV cache) in non-contiguous pages.

Specifically, by storing KV cache in non-contiguous pages, the model avoids the wasted space that would occur if it were stored contiguously (e.g., in a single contiguous block), which would consume most of the available GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps when **prefill is compute-bound and decode is memory-bandwidth-bound**, as stated in the context.

This is because the context explicitly explains that:
1.  **Prefill** is compute-bound (requires significant processing power).
2.  **Decode** is memory-bandwidth-bound (requires significant data transfer speed).

By separating these steps, the system can utilize di


## Which N16-N19 pieces are real

### Breakdown of components:
- **N16 (Embedding & Chunking):** Stubbed (fallback keyword matching, 0.0 ms).
- **N17 (Vector Index / Retrieval):** Stubbed (simple keyword overlap score on in-memory dictionary, 0.1 ms).
- **N18 (Reranker):** Stubbed (sorted by toy keyword score).
- **N19 (Lakehouse / Data storage):** Stubbed (in-memory `TOY_DOCS` Python list).
- **LLM Serving:** **REAL** (live local `llama-server` HTTP endpoint on port 8080 executing Qwen 0.8B Q4_K_M).

### Latency analysis & Optimization:
The dominant stage is overwhelmingly **llm** (7439.3 ms out of 7439.5 ms, 100.0% of total pipeline latency). This precisely matches theoretical expectations: in-memory toy retrieval is microsecond-scale, whereas LLM autoregressive token generation on CPU is strictly memory-bandwidth-bound (~40 ms per output token).

To halve this pipeline's latency (from ~7.4s down to ~3.7s), the only effective attack vector is the **LLM generation stage**:
1. **Reduce `max_tokens`:** Restricting the answer length to ~50 tokens immediately cuts decode latency by over 50%.
2. **Optimal thread setting:** Applying `LAB_N_THREADS=7` (found in CP2) delivers a 1.13x decode speedup by avoiding E-core synchronization drag.
3. **Prompt caching:** Since the system prompt is identical across all requests, enabling Radix/prefix caching avoids recalculating prefill for shared tokens.
