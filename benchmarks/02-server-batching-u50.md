# 02 - Continuous batching under load (u50)

Host `Darwin-arm64` · `--parallel 4` · 30 samples over
60s at 2.0s intervals · raw CSV: `02-server-metrics-u50.csv`

| Gauge | Peak observed |
|:--|--:|
| `n_busy_slots_per_decode` (avg/decode) | 3.85 of 4 slots (96%) |
| `requests_processing` | 4 |
| `requests_deferred` | 46 |
| `kv_cache_usage_ratio` | n/a — not exported by llama.cpp `b10488` |
| `tokens_predicted_total` (final) | 6552 |

Highest sampled value was **3.85 of 4** slots. Note this gauge is llama.cpp's *average* busy slots per decode step, so the number below is the highest average we sampled, not an instantaneous maximum batch width. A peak near 1 means
requests were served one at a time -- either the load was too light to overlap, or
they arrived too far apart. A peak approaching `--parallel` means the scheduler was
genuinely packing concurrent requests into shared decode steps.
`requests_deferred` went above zero: more requests arrived than there were slots, so some waited. That wait is the queue time in your P95.

## Your observation

Peak `n_busy_slots_per_decode` was **3.85 of 4 slots (96%)**. `requests_processing` sat at 4 the whole minute and `requests_deferred` peaked at **46**. Continuous batching is packing the decode step; the scheduler is not serializing.

That 3.85 does **not** match Little's-Law effective concurrency of **28.0** in `02-server-results.md`, and both numbers are right. 3.85 is *slot utilisation* (how many of the 4 decode slots were busy per step). 28.0 is *occupancy including the queue*. The gap (~24 requests) is wait time, which is exactly what `requests_deferred ≈ 46` is showing as an instantaneous gauge. I trust 3.85 for "is batching on?" and 28.0 for "are we saturated?".
