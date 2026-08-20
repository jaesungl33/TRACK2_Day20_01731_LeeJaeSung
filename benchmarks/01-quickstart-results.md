# 01 - Measure: latency baseline

Model `Gemma 4 E2B` · host `Darwin-arm64` · llama.cpp `b10488`
Settings: `threads=10` `ngl=99` `ctx=2048`
`max_tokens=64` · warm-up discarded
Completed requests: `UD-Q4_K_XL` 10/10 · `UD-Q2_K_XL` 10/10

| Quantization | Size (GB) | Load (ms) | TTFT P50/P95 (ms) | TPOT P50/P95 (ms) | E2E P50/P95/P99 (ms) | Decode (tok/s) |
|:--|--:|--:|--:|--:|--:|--:|
| UD-Q4_K_XL | 2.97 | 5098 | 121 / 258 | 23.1 / 33.1 | 1570 / 2231 / 2231 | 43.3 |
| UD-Q2_K_XL | 2.24 | 3033 | 116 / 485 | 18.9 / 19.9 | 1282 / 1736 / 1736 | 52.8 |

- **TTFT** = prefill. Short prompts keep it small; long-context RAG is where it explodes.
- **TPOT** = per-output-token decode cost, bounded by memory bandwidth. `decode tok/s = 1000 / TPOT_p50`.
- `UD-Q2_K_XL` decodes **1.22x faster** than `UD-Q4_K_XL` here, for 0.73 GB less on disk.

## Your observation

UD-Q2_K_XL decodes **1.22×** faster (52.8 vs 43.3 tok/s) and is **0.73 GB** smaller (2.24 vs 2.97 GB, −25%). Load time also dropped 5.1 s → 3.0 s. TTFT P50 is almost identical (116 vs 121 ms); the 2-bit TTFT P95 (485 ms) is a first-request outlier, not a new prefill regime.

I asked both servers the same question (*"what is PagedAttention and why does it matter for serving?"*). 4-bit named KV-cache pages and memory fragmentation. 2-bit stayed generic ("memory blocks", "faster inference") and never said KV cache. On a 16 GB M4 I keep **UD-Q4_K_XL**: a 22% decode bump is not worth the precision loss for a serving lab that has to explain itself.
