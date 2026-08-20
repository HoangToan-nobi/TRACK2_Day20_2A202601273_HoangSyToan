# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Hoàng Sỹ Toàn
**Cohort:** A20-K2
**Ngày submit:** 2026-08-21

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** Linux 7.0.0-29-generic (x86_64)
- **CPU:** 12th Gen Intel Core i9-12900H
- **Cores:** 14 physical · 20 logical
- **CPU extensions:** AVX2
- **RAM:** 15.3 GB
- **Accelerator:** CPU only (no GPU detected)
- **llama.cpp asset đã tải:** llama-b10488-bin-ubuntu-x64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL (primary) + UD-Q2_K_XL (compare)

**Chạy ở đâu:** Laptop của tôi (Linux x86_64, 15.3 GB RAM)

**Setup story:** Setup ran smoothly — no special configuration needed. Machine had 15.3 GB RAM, exceeding the 8 GB requirement. `make probe` detected AVX2 extension support, allowing the prebuilt binary to run efficiently. Both model downloads completed without interruption. Only change: confirmed the chosen model was Gemma 4 E2B (default) via `models/active.json`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 4123 | 195 / 236 | 54.7 / 56.3 | 3623 / 3731 / 3731 | 18.3 |
| UD-Q2_K_XL | 2.24 | 3065 | 272 / 319 | 41.9 / 42.7 | 2881 / 3001 / 3001 | 23.9 |

**Quan sát:** On this machine (Intel i9-12900H, 15.3 GB RAM, CPU-only), **UD-Q2_K_XL is the better choice**. The 2-bit quantization decodes 1.31× faster (23.9 vs 18.3 tok/s, or 41.9 vs 54.7 ms TPOT) and saves 0.73 GB on disk. The 77 ms TTFT penalty (272 vs 195 ms) is acceptable because prefill happens once per request while decode happens for every output token—users notice decode speed much more. The quality difference is subtle and both quantizations produce coherent, reasonable responses on similar prompts.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 0.69 | 12000 | 22000 | 23000 | 8.7 | 0.0% |
| 50 | 0.90 | 28000 | 53000 | 56000 | 25.1 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 1.30×
- **P95 tăng:** 2.41×
- **Effective concurrency ở 50 users:** 25.1 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang chạy): 3.94 / 4 slots

**Saturation reading:** Saturation point: between 10 and 50 users (somewhere closer to 10). The evidence: **effective concurrency = 25.1 at 50 users vs. only 4 parallel slots**. This occupancy/slot ratio of 6.28 means the server queue is 6.3× longer than the number of requests it can process in parallel—classic saturation. Secondary evidence: P95 jumped 2.41× (22→53 sec) while throughput only rose 1.30× (0.69→0.90 RPS), the hallmark of a queue-bound system. To raise goodput@SLO=P95<30s: first, increase `--parallel` from 4 to 8 slots (doubles request-in-flight capacity without increasing per-request latency).

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | — | stub (localhost only) |
| N17 Data pipeline | — | stub (in-memory list) |
| N18 Lakehouse | — | stub (keyword dict) |
| N19 Vector + features | — | stub (keyword overlap, no embeddings) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms (keyword overlap fallback)
- llm: 2295.2 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection:** Bottleneck is entirely in the LLM stage (2295 ms), which matches expectations for token generation on a CPU-bound model without streaming output. The embedding and retrieval stubs are intentionally fast (0 ms keyword overlap). To reduce pipeline latency 2×, attack the LLM: switch to UD-Q2_K_XL quantization (31% faster decode) or increase `--parallel` to reduce queue time. Embedding would help if real, but keyword overlap is already optimal for a stub.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** Thread count sweep: found optimal at 14 threads (physical cores), tested against 20 threads (all logical cores including hyperthreads)

```
before:  20 threads @ 17.3 tok/s
after:   14 threads @ 19.1 tok/s
speedup: 1.10×
```

**Tại sao nó work:** The speedup is driven by **memory bandwidth contention, not CPU compute**. The i9-12900H has 14 physical cores sharing one L3 cache (20 MB) and memory controllers. When using 20 threads (including hyperthreads on 6 cores), the extra threads (15–20) compete with their siblings for the same cache lines, prefetch bandwidth, and memory access slots. Since decode is bandwidth-bound—moving matrix rows from DRAM into CPU cache, not doing heavy computation—adding threads past physical cores adds memory latency and cache coherency overhead without increasing available bandwidth. The 2.84× difference between 1-thread (6.7 tok/s) and 14-thread (19.1 tok/s) also confirms: we're not CPU-saturated (if we were, logical threads would help equally). The knee at exactly physical-core count is the clearest proof the bottleneck is memory, not compute.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** _<B1 build-compare / B2 sweep nào / B4 challenge nào / B5 lựa chọn nào>_

**Numbers:**

```
before:  <số>
after:   <số>
speedup: <X.Y>×
```

**Điều này nói lên gì mà deck chưa nói:**

_(để trống nếu bạn không làm phần này)_

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

The 2-bit quantization actually outpaces the 4-bit on decode speed by 31%, which is huge given that LLMs are usually compute-bound. But it makes sense under closer inspection: decode is memory-bandwidth-bound here (on CPU), so fewer bytes per token (Q2 vs Q4) directly translates to faster memory throughput. The real surprise is that this CPU i9-12900H can achieve 23.9 tok/s with Q2, which rivals some GPU inference speeds from a year ago—the engineering in llama.cpp to vectorize across AVX2 is genuinely impressive.

---

## 8. Self-check trước khi push

- [ ] `hardware.json` committed
- [ ] `models/active.json` committed
- [ ] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [ ] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [ ] `benchmarks/02-server-results.md` committed (`make load-report`)
- [ ] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [ ] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [ ] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [ ] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [ ] 5 screenshots trong `submission/screenshots/`
- [ ] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [ ] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
