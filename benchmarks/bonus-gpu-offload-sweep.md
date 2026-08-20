# Bonus - GPU offload sweep

Host `Darwin-arm64` · backend(s) `apple_metal` ·
llama.cpp `b10488` · `threads=1` · metric `tg128`

| -ngl | tg128 (tok/s) | vs -ngl 0 | vs best |
|:--|--:|--:|--:|
| 0 | 17.2 | 1.00x | 43% |
| 8 | 20.2 | 1.18x | 50% |
| 16 | 21.6 | 1.26x | 54% |
| 24 | 26.4 | 1.54x | 65% |
| 32 | 28.9 | 1.68x | 71% |
| 99 | 40.4 | 2.35x | 100% |

Best: `-ngl 99` at 40.4 tok/s
-- 2.35x faster than CPU-only.

Where the curve flattens tells you the model ran out of layers to move. Where it
*peaks below* full offload tells you something did not fit and the accelerator
started paying to fetch weights it could not hold.

## Your finding

Full offload is best: the curve never peaks early. `-ngl 0` (CPU) is 17.2 tok/s; `-ngl 32` is still only 71% of peak; `-ngl 99` is **40.4 tok/s, 2.35× vs CPU**. Gemma 4 E2B has 35 layers, so 32 is still a partial split — the last jump is those remaining layers leaving the CPU.

Nothing ran out of VRAM. M4 16 GB is **unified memory**; 3 GB of Q4 weights plus a 2048-token KV cache sit in the same pool the GPU already uses. The "partial offload because the model does not fit" story from discrete NVIDIA cards does not apply here. Keep `-ngl 99`. The host↔device copy tax that makes partial offload interesting on PCIe is mostly gone on Apple Silicon.
