# Reflection — Day 20 Lab (Personal Report)

> **Đây là báo cáo cá nhân.** Số liệu của bạn **không** so sánh được với bạn cùng lớp
> — chỉ so **before vs after trên chính máy bạn**. Rubric chấm độ rõ ràng của setup,
> đo lường và **lập luận**, không chấm tốc độ tuyệt đối.
>
> `make verify` sẽ fail nếu còn placeholder chưa điền. Đó là cố ý.

**Họ Tên:** Lee Jae Sung
**Cohort:** A20-K1
**Ngày submit:** 2026-08-20

---

## 1. Hardware & runtime  *(rubric 1, 2 — 10 điểm)*

> Từ `make probe`. Paste output hoặc điền tay.

- **OS:** macOS 26.5.2 (Darwin 25.5.0 arm64)
- **CPU:** Apple M4
- **Cores:** 10 physical / 10 logical
- **CPU extensions:** NEON
- **RAM:** 16.0 GB
- **Accelerator:** Apple Metal (MTL0: Apple M4, 12124 MiB)
- **llama.cpp asset đã tải:** llama-b10488-bin-macos-arm64.tar.gz
- **Model đã dùng:** Gemma 4 E2B (`LAB_MODEL=gemma4-e2b`)
- **Quantization:** UD-Q4_K_XL + UD-Q2_K_XL (từ `models/active.json`)

**Chạy ở đâu:** laptop của tôi
_(Nếu dùng cloud fallback: nói rõ vì sao — RAM < 8 GB, setup fail, v.v. Không mất điểm.)_

**Setup story** (≤ 80 chữ): điều gì cần thay đổi để lab chạy trên máy bạn? Có bước
nào fail rồi phải workaround không?

macOS ships `/usr/bin/python3` 3.9.6; the lab needs ≥ 3.10. I created `.venv` with Homebrew Python 3.11.14, then `make setup`. Hugging Face and GitHub both worked; no mirror. 16 GB RAM was enough for default Gemma 4 E2B. After `make tune` I served with `LAB_N_THREADS=1`.

---

## 2. Đo lường  *(rubric 3, 4, 5 — 20 điểm)*

> Paste bảng từ `benchmarks/01-quickstart-results.md` (`make bench` tự sinh).

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|---|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5098 | 121 / 258 | 23.1 / 33.1 | 1570 / 2231 / 2231 | 43.3 |
| UD-Q2_K_XL | 2.24 | 3033 | 116 / 485 | 18.9 / 19.9 | 1282 / 1736 / 1736 | 52.8 |

**Quan sát** (≤ 60 chữ): 2-bit nhanh hơn bao nhiêu, và **có đáng không**? Bạn đã thử
hỏi cùng một câu trên cả hai (`make serve` vs `.venv/bin/python labs/02-serve/serve.py --compare`)
chưa? Chất lượng khác nhau thế nào?

2-bit decodes **1.22×** faster (52.8 vs 43.3 tok/s) and is 0.73 GB smaller. Same PagedAttention prompt: 4-bit named KV-cache fragmentation; 2-bit said only "memory blocks". Not worth it on 16 GB — I keep Q4.

---

## 3. Serving under load  *(rubric 8, 9, 10 — 20 điểm)*

> Từ `benchmarks/02-server-results.md` (`make load-report`).

| Users | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|--:|--:|--:|--:|--:|--:|--:|
| 10 | 1.09 | 7500 | 11000 | 19000 | 8.4 | 0.0% |
| 50 | 1.02 | 29000 | 50000 | 52000 | 28.0 | 0.0% |

- **Offered load tăng 5×, throughput thực tăng:** 0.94×
- **P95 tăng:** 4.55×
- **Effective concurrency ở 50 users:** 28.0 so với `--parallel` = 4 slots

**Peak `llamacpp:n_busy_slots_per_decode`** (từ `make metrics` khi `make load-50` đang
chạy): 3.85 / 4 slots

**Saturation reading** (≤ 80 chữ): server của bạn bão hoà ở đâu, và **bằng chứng nào**
thuyết phục bạn? Nếu P95 tăng nhanh hơn RPS thì phần latency thêm đó là queue time hay
compute time — bạn biết bằng cách nào? Nếu bạn phải nâng goodput@SLO, bạn sẽ đổi knob
nào **trước**, và vì sao knob đó?

Saturated at or below 10 users: Little's Law already gives 8.4 in-flight vs 4 slots. 5× offered load bought 0.94× RPS and 4.55× P95. The extra P95 is **queue time** — busy_slots 3.85/4 and deferred=46 mean compute is full. First knob: raise `--parallel` (occupancy/slot = 7.01 is wait, not decode). Watch KV: 8 slots cut ctx/slot to 256.

---

## 4. Integration  *(rubric 12, 13 — 15 điểm)*

> Từ `make pipeline`. Nói thật cái nào real, cái nào stub — stub **không** mất điểm.

| Day | Piece | Real hay stub? |
|---|---|---|
| N16 Cloud/IaC | k8s / Compose | stub (localhost only) |
| N17 Data pipeline | Airflow / batch | stub (in-memory list) |
| N18 Lakehouse | Delta / Iceberg | stub (toy dict `TOY_DOCS`) |
| N19 Vector + features | vector index + Feast | stub (keyword overlap) |
| N20 Serving | `llama-server` | real |

**Latency split** (mean của 3 query, từ output của `pipeline.py`):

- embed: 0.0 ms
- retrieve: 0.0 ms
- llm: 985.9 ms
- **stage chiếm nhiều nhất:** llm (100% của total)

**Reflection** (≤ 60 chữ): bottleneck ở đâu? Có khớp với kỳ vọng của bạn không? Nếu
phải giảm latency của pipeline này 2×, bạn sẽ tấn công vào đâu?

LLM is 100% of 986 ms. Expected: 6-doc keyword retrieve cannot beat ~25 ms/tok decode. To cut 2× I attack decode (shorter `max_tokens`, prefix-cache the system prompt, or 2-bit). Embed/retrieve are already zero.

---

## 5. The single change that mattered most  *(rubric 11 — 10 điểm)*

> **Phần quan trọng nhất của report.** Không cần bonus track: `make tune` đã cho bạn
> một before/after thật (`benchmarks/01-tuning-tg128.md`). Đổi quantization,
> `LAB_N_CTX`, hay `--parallel` rồi đo lại cũng được.

**Change:** drop `-t` from the physical-core default of 10 down to 1 (`LAB_N_THREADS=1`)

```
before:  42.1 tok/s  (tg128, -t 10, ngl=99)
after:   47.2 tok/s  (tg128, -t 1,  ngl=99)
speedup: 1.12×
```

**Tại sao nó work** (1–2 đoạn — đây là phần grader đọc kỹ nhất):

The deck's expected curve is "throughput climbs to physical cores, then drops." Mine did the opposite: 47.2 → 45.8 → 42.1 → 40.5 tok/s as threads went 1 → 5 → 10 → 20. The knee is at **one** thread, 12% above the default that `hardware.json` would have picked.

The mechanism is where the work actually runs. `make probe` reported Metal and the runtime set `ngl=99`, so every Gemma layer lives on the GPU. Decode is then a stream of weight reads through GPU memory bandwidth, not a CPU GEMM. Extra CPU threads cannot issue more of those reads; they only add host-side scheduling and Metal command-queue sync. Oversubscribe to 20 threads (2× logical) and that overhead shows up as a 14% loss versus `-t 1`.

This is the same bandwidth-bound story as the deck, just on the other device. On CPU-only, you stop adding threads when they start fighting for DRAM channels. On Metal with full offload, you stop at one host thread because the DRAM channels that matter are already inside the GPU. I served the load tests with `LAB_N_THREADS=1` for that reason. A 1.12× decode gain is small in isolation; the useful lesson is not to copy the physical-core default once an accelerator owns the matmuls.

---

## 6. Bonus  *(optional — tối đa 20 điểm)*

> Bỏ trống nếu không làm. Xem `bonus/README.md`. Đừng làm hết — **một** finding sâu
> ăn điểm hơn năm bảng nông.

**Đã làm:** B2 `make sweep-gpu` · B3 below (from B2, not from `make tune`) · B4 challenge C2 (quantized KV cache, `bonus/C2-kv-cache.md`) · B5/C8 semantic cache (`benchmarks/bonus-semantic-cache.md`). B1 skipped: cmake is not installed on this Mac.

**Numbers** (B2 GPU offload, same model, same `-t 1`, only `-ngl` changes):

```
before:  17.2 tok/s  (tg128, -ngl 0, CPU)
after:   40.4 tok/s  (tg128, -ngl 99, Metal)
speedup: 2.35×
```

C2 extra: process RSS at `n_ctx=4096` dropped 3168 MB (f16 KV) → 2357 MB (q8_0 KV), **−811 MB (−26%)**.

**Điều này nói lên gì mà deck chưa nói:**

The deck's GPU-offload story is "move layers until VRAM is full, then stop." On this M4 the curve never flattened: `-ngl 32` was still only 71% of peak because Gemma 4 E2B has 35 layers, and 16 GB unified memory holds the whole 3 GB Q4 model plus KV. There is no PCIe tax and no "does not fit" knee, so the right answer is just `-ngl 99`. The 2.35× vs CPU is the largest speedup I measured in the lab — larger than the 1.12× from thread count — and it comes from putting decode on the memory bus that actually feeds the GPU, not from compiling a native binary.

C2 is the same bytes-per-token idea on the KV, not the weights. q8_0 KV is the knob I would actually turn to buy more `--parallel` slots on unified memory. C8 (offline) showed why I will not quote a semantic-cache hit rate from a bag-of-words stub: "TTFT" vs "time to first token" scored 0.00 (false miss), the threshold sweep was flat at 3/8, and no single threshold can fix a collapsed embedder.

---

## 7. Điều làm bạn ngạc nhiên nhất  *(optional)*

The thread sweep peaking at `-t 1` on a 10-core M4. I came in expecting the knee at 10. Full Metal offload made the CPU core count almost irrelevant for decode, and the "more threads" instinct was actively slower.

---

## 8. Self-check trước khi push

- [x] `hardware.json` committed
- [x] `models/active.json` committed
- [x] `benchmarks/01-quickstart-results.md` committed (`make bench`)
- [x] `benchmarks/01-tuning-tg128.md` committed (`make tune`)
- [x] `benchmarks/02-server-results.md` committed (`make load-report`)
- [x] `benchmarks/02-server-batching-u50.md` hoặc `-metrics-u50.csv` committed (`make metrics`)
- [x] `benchmarks/locust-10_stats.csv` + `locust-50_stats.csv` committed (`make load-10` / `load-50`)
- [x] `benchmarks/03-integration-results.md` committed (`make pipeline`)
- [x] Mọi section **"required — replace this line"** trong các file `benchmarks/*.md`
      đã được thay bằng nhận xét của bạn
- [x] 5 screenshots trong `submission/screenshots/`
- [x] `make verify` → **exit 0**
- [ ] Repo GitHub ở chế độ **public**
- [ ] Đã paste public URL vào VinUni LMS
- [x] **Không** commit `models/*.gguf` hay `runtime/` (đã có trong `.gitignore`)

**Quan trọng:** repo phải **public** đến khi điểm được công bố. Private → grader không
xem được → 0 điểm.
