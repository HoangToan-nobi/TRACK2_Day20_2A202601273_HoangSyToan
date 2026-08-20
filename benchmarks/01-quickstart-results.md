# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Linux-x86_64` · llama.cpp `b10488`
Settings: `threads=14` `ngl=0` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4123 | 195 / 236 | 54.7 / 56.3 | 3623 / 3731 / 3731 | 18.3 |
| UD-Q2_K_XL | 2.24 | 3065 | 272 / 319 | 41.9 / 42.7 | 2881 / 3001 / 3001 | 23.9 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.31x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

On this machine (Intel i9-12900H, 15.3 GB RAM, CPU-only), **UD-Q2_K_XL is the better choice**. The 2-bit quantization decodes 1.31× faster (23.9 vs 18.3 tok/s, or 41.9 vs 54.7 ms TPOT) and saves 0.73 GB on disk. The 77 ms TTFT penalty (272 vs 195 ms) is acceptable because prefill happens once per request while decode happens for every output token—users notice decode speed much more. The quality difference is subtle and both quantizations produce coherent, reasonable responses on similar prompts.
