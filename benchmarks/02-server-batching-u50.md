# 02 - Continuous batching under load (u50)

Host `Linux-x86_64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.94 of 4 slots (99%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 5199 |

Highest sampled value was **3.94 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak batch width was **3.94 of 4 slots** (99% utilization). This matches the effective concurrency story in `02-server-results.md`—effective concurrency at 50 users was 25.1, meaning ~25 requests in flight total, but only 3.94 are actively being decoded per step. The remaining ~21 requests are queued waiting for a slot to free up. I trust the `n_busy_slots_per_decode` metric (3.94) for showing batch width since it directly measures what's happening inside the scheduler each step. The effective concurrency (25.1) is higher because it includes all requests in flight (processing + queued), using Little's Law. Both numbers are consistent: high effective concurrency + high batch width confirms the server is CPU-bound on decode, not memory-bound on the client side.
