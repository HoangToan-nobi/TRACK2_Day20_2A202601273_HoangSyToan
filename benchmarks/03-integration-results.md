# 03 - Integrate: RAG pipeline run

Host `Linux-x86_64` · llama.cpp `b10488` ·
retrieval backend: **keyword overlap** · 3 queries

| Query | Contexts retrieved | embed (ms) | retrieve (ms) | llm (ms) | total (ms) |
|:--|--:|--:|--:|--:|--:|
| Why is goodput more useful than raw throughp... | goodput, paged, radix | 0.0 | 0.1 | 2672.5 | 2672.6 |
| What problem does PagedAttention actually so... | paged, radix, disagg | 0.0 | 0.0 | 2079.9 | 2080.0 |
| When does splitting prefill and decode help?... | disagg, radix, batching | 0.0 | 0.0 | 2133.2 | 2133.3 |

Mean per stage (ms): embed **0.0** · retrieve **0.0** ·
llm **2295.2** · total **2295.3**
Dominant stage: **llm** (100% of total)

## Answers returned

**Why is goodput more useful than raw throughput?**

> Goodput@SLO counts only the requests per second that met the TTFT and TPOT targets. Throughput at saturation ignores SLOs.

**What problem does PagedAttention actually solve?**

> PagedAttention stores the KV cache in non-contiguous pages, removing the internal fragmentation that wasted most GPU memory.

**When does splitting prefill and decode help?**

> Splitting prefill and decode helps because prefill is compute-bound and decode is memory-bandwidth-bound.


## Which N16-N19 pieces are real

- **N16 Cloud/IaC**: Stub — localhost-only, no orchestration
- **N17 Data pipeline**: Stub — in-memory list of toy documents
- **N18 Lakehouse**: Stub — dict-based "index" with keyword keys
- **N19 Vector + features**: Stub — keyword overlap, no actual embeddings (embedding server not running)
- **N20 Serving**: Real — actual `llama-server` endpoint

The LLM stage dominating (2295 ms = 100%) is expected because token generation on CPU is slow compared to retrieval. To halve pipeline latency, attack the LLM: switch to Q2 quantization (31% faster decode, saves ~715 ms to ~1578 ms total) or increase `--parallel` to reduce queue time. Embedding + retrieval together are 0 ms in stub mode, so optimizing them here gives no real gain until they become real (at which point a vector search backend would be the next bottleneck).
