# 03 - Integrate: RAG pipeline run

Host `Darwin-arm64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.0 | 1186.9 | 1187.0 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 873.1 | 873.1 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 897.7 | 897.7 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **985.9** · total **985.9**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, which removes the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

| Day | Piece | Real or stub? |
|---|---|---|
| N16 Cloud/IaC | k8s / Compose | **stub** — localhost only |
| N17 Data pipeline | Airflow / batch job | **stub** — in-memory list |
| N18 Lakehouse | Delta / Iceberg | **stub** — toy dict `TOY_DOCS` |
| N19 Vector + features | vector index + Feast | **stub** — keyword overlap, no embedder (`embed: 0.0 ms`) |
| N20 Serving | `llama-server` | **real** — Gemma 4 E2B UD-Q4_K_XL on `:8080` |

Dominant stage is **llm at 100%** of 985.9 ms mean. Expected: retrieval is a 6-doc keyword scan, so it cannot compete with ~25 ms/tok decode. To halve this pipeline I would attack decode (lower `max_tokens`, keep the system prompt byte-identical for prefix cache, or switch to 2-bit). Embed/retrieve are already 0.0 ms — they have nothing to give.
