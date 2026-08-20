# 02 - Serve: load test + saturation reading

Host `Darwin-arm64` · llama.cpp `b10488` ·
`--parallel 4` · `ctx=2048` · `threads=1` (LAB_N_THREADS from `make tune`) ·
`ngl=99`

| Users | Reqs | RPS | P50 (ms) | P95 (ms) | P99 (ms) | Eff. concurrency | Failures |
|:--|--:|--:|--:|--:|--:|--:|--:|
| 10 | 61 | 1.09 | 7500 | 11000 | 19000 | 8.4 | 0.0% |
| 50 | 58 | 1.02 | 29000 | 50000 | 52000 | 28.0 | 0.0% |

*Effective concurrency = RPS x average latency (Little's Law) -- how many requests were
really in flight, regardless of how many users locust simulated. It counts queued requests
too, so the occupancy/slot ratio can legitimately exceed 1.0; it is occupancy, not
utilisation. For true slot utilisation use the server's own gauges (`make metrics`).*

## What these two runs say

| Going from 10 to 50 users | |
|:--|--:|
| Offered load | 5x |
| Throughput actually delivered | **0.94x** (19% of linear) |
| P95 latency | **4.55x** |
| Effective concurrency at 50 users | 28.0 vs `--parallel 4` slots (occupancy/slot ratio 7.01) |

**Saturated.** Throughput delivered only 0.94x for 5x the offered load, and effective concurrency (28.0) is at or above all 4 decode slots. Saturation sets in somewhere at or below 50 users; the load you added beyond that point became queue time rather than throughput.

Throughput moved 0.94x while P95 moved 4.55x. That gap is the goodput argument: past saturation you buy throughput by spending latency, and if your SLO is a P95 target then the requests you added are no longer being served within it. (This lab does not fix an SLO number for you -- pick one in your write-up and state how much goodput you keep at it.)

## Your reading

Saturated **at or below 10 users**, not at 50. The number that convinced me: at 10 users, effective concurrency is already **8.4 vs 4 slots**, so arrivals were already queueing. Raising offered load 5× (10 → 50 users) delivered **0.94×** throughput (1.09 → 1.02 RPS) while P95 jumped **4.55×** (11 s → 50 s). That extra P95 is queue time: `n_busy_slots_per_decode` peaked at 3.85/4 and `requests_deferred` hit 46, so compute was full and the new users waited.

If I had to raise goodput@SLO (say P95 ≤ 8 s), the first knob is **`--parallel`**, not more threads and not more users. Occupancy/slot is 7.01, so most of the in-flight population is queued. More slots cut wait. I would not raise `--parallel` blindly: `n_ctx=2048` is shared, so 8 slots drop per-slot context to 256 tokens and RAG prompts will truncate. Second choice if RAM for KV cannot grow: shrink `max_tokens` so each slot turns over faster and the same 4 slots drain the queue.
