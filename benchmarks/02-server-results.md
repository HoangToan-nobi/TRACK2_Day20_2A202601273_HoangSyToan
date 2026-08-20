# 02 - Serve: load test + saturation reading

Host `Linux-x86_64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=14` ·
`ngl=0`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 40 | 0.69 | 12000 | 22000 | 23000 | 8.7 | 0.0% |
| 50 | 51 | 0.90 | 28000 | 53000 | 56000 | 25.1 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **1.30x** (26% of linear) |
| P95 latency | **2.41x** |
| Effective concurrency at 50 users | 25.1 vs `--parallel 4` slots (occupancy/slot ratio 6.28) |

**Saturated.** Throughput delivered only 1.30x for 5x the offered load, and effective concurrency (25.1) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 1.30x while P95 moved 2.41x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

**Saturation point: between 10 and 50 users** (somewhere closer to 10). The evidence: **effective concurrency = 25.1 at 50 users vs. only 4 parallel slots**. This occupancy/slot ratio of 6.28 means the server queue is 6.3× longer than the number of requests it can process in parallel—classic saturation. Secondary evidence: P95 jumped 2.41× (22→53 sec) while throughput only rose 1.30× (0.69→0.90 RPS), the hallmark of a queue-bound system. 

To raise goodput@SLO=P95<30s: first, increase `--parallel` from 4 to 8 slots (doubles request-in-flight capacity without increasing per-request latency). This addresses the queue depth directly. Second, if that's insufficient, move up to the Q2 quantization (23.9 tok/s vs 18.3 tok/s) to reduce TPOT by 31%, cutting P95 latency proportionally. Adding threads past 14 would hurt (already at memory bandwidth knee); reducing context would cut queue depth but break RAG use cases.
