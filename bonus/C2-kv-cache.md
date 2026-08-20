# Bonus C2 — Quantized KV cache (`q8_0` vs default `f16`)

Host: Darwin-arm64 · Apple M4 · 16 GB unified memory · llama.cpp `b10488`
Model: Gemma 4 E2B `UD-Q4_K_XL` · `threads=1` · `--parallel 4` · `n_ctx=4096` (1024 tokens/slot)

This is the laptop version of the deck's "FP8 KV cache" slide. llama.cpp exposes it as `--cache-type-k` / `--cache-type-v`. Allowed types on this binary include `f16` (default) and `q8_0`.

## Setup

Two sequential `llama-server` processes, same model, same flags except KV dtype. After `/health` was ok, one 48-token completion was sent so KV pages were actually allocated, then RSS was read from `ps`.

```
LAB_N_THREADS=1 LAB_N_CTX=4096 .venv/bin/python labs/02-serve/serve.py
LAB_N_THREADS=1 LAB_N_CTX=4096 .venv/bin/python labs/02-serve/serve.py -- --cache-type-k q8_0 --cache-type-v q8_0
```

## Numbers

| KV dtype | Process RSS after 1 completion | vs f16 |
|---|--:|--:|
| f16 (default) | 3168.0 MB | 1.00× |
| q8_0 | 2357.4 MB | **−810.6 MB (−25.6%)** |

The 3 GB of Q4 weights do not change between runs, so almost all of that 811 MB is KV (and Metal's f16 scratch) going away. Doubling `n_ctx` from the lab default 2048 to 4096 was intentional: at short context the KV is a rounding error next to the weights, which would hide the knob the challenge is about.

## Quality / latency

I did not run a 10-prompt JSON-extraction eval. q8_0 on K and V is an 8-bit integer cache, not 2-bit weights; for this model the answers to the one shared prompt were still coherent. The trade I would actually take on this laptop: **q8_0 KV, keep UD-Q4_K_XL weights**. Saving ~0.8 GB at 4k context is real headroom if I later raise `--parallel` or `--ctx-size`. I would not drop to `q4_0` KV without an eval — that is the same "small and maybe broken" trade I already rejected for 2-bit *weights*.

## What this says that the deck's FP8 slide does not

On a datacenter GPU, FP8 KV is about fitting a longer context into HBM. On unified-memory Apple Silicon the same knob is about leaving RAM for the OS and for more `--parallel` slots. 811 MB is roughly another 4-bit copy of this model; that is the difference between "I can raise `--parallel` from 4 to 8" and "I start swapping". The mechanism is identical (fewer bytes per cached token). The constraint that moves is not HBM, it is the one pool the GPU and the browser share.
