# 01 - Tune: thread-count sweep

Model `gemma-4-E2B-it-UD-Q4_K_XL.gguf` · host `Darwin-arm64` · llama.cpp `b10488`
CPU: **10 physical · 10 logical** cores · `ngl=99` · metric `tg128`

| threads (-t) | tg128 (tok/s) | vs best |
|:--|--:|--:|
| 1 | 47.2 | 100% |
| 5 | 45.8 | 97% |
| 10 | 42.1 | 89% |
| 20 | 40.5 | 86% |

**Best**: `-t 1` at 47.2 tok/s
**Slowest tested**: `-t 20` at 40.5 tok/s (1.17x spread)
**Against the physical-core default** (`-t 10`, 42.1 tok/s): 1.12x

Use this in your run:

```bash
LAB_N_THREADS=1 make bench
```

## Your explanation

The knee is at **`-t 1`**, not at the 10 physical cores. Throughput falls monotonically: 47.2 → 45.8 → 42.1 → 40.5 tok/s as threads go 1 → 5 → 10 → 20. That is the opposite of the CPU-only curve in the deck.

Cause: `ngl=99` put every layer on Metal. Decode is then bound by GPU memory bandwidth, not by CPU FLOPs. Extra host threads do not feed the GPU more weights; they add scheduling and Metal-command-queue sync on the CPU side. Oversubscribing to 20 threads makes that overhead worse (86% of peak). The "set `-t` to physical cores" heuristic is for CPU decode. Once the accelerator owns the matmuls, fewer host threads is the right default.
