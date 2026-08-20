# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Linux-x86_64` · llama.cpp `b10488`
CPU: **14 physical · 20 logical** cores · `ngl=0` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 6.7 | 35% |
| 7 | 16.7 | 88% |
| 14 | 19.1 | 100% |
| 20 | 17.3 | 91% |
| 40 | 14.4 | 76% |

**Best**: `-t 14` at 19.1 tok/s
**Slowest tested**: `-t 1` at 6.7 tok/s (2.84x spread)
**Against the physical-core default** (`-t 14`, 19.1 tok/s): 1.00x

Use this in your run:

```bash
LAB_N_THREADS=14 make bench
```

## Your explanation

**The knee is exactly at 14 physical cores**, as expected. Beyond that, throughput drops: 20 threads (hyperthreads) hit 91%, then 40 threads fall to 76%. The reason: decode is **memory-bandwidth-bound**, not compute-bound. The physical cores share a fixed L3 cache (20 MB) and memory controllers; hyperthreads on the same core contend for the same cache lines and prefetch bandwidth. Using logical cores 15–20 (hyperthreads of physical 1–6) adds context switches and coherency overhead without increasing memory throughput, so they slow the decode loop. The fact that peak stays at exactly the physical core count confirms the bottleneck: with more compute available but no more bandwidth, adding threads hurts.
