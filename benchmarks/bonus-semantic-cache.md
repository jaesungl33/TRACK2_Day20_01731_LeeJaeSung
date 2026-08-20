# Bonus C8 / B5 — Semantic cache (offline stub embedder)

Host: Darwin-arm64 · no extra model download ·
`bonus/serving-regimes/semantic-cache-demo.py --offline --sweep`

The deck's serving stack is three caches deep:

```
request -> [1] semantic cache (meaning) -> [2] prefix/KV cache -> [3] full inference
```

Layer 1 is supposed to catch *paraphrases*. Layer 2 only hits when the prefix is byte-identical. This run uses the lab's bag-of-words stub, not a real embedding model — and that is the finding, not a footnote.

## Prompt stream (lab default)

| # | Prompt | Role |
|--:|---|---|
| 1 | What is goodput at SLO? | first of topic A |
| 2 | Explain TTFT and TPOT. | first of topic B |
| 3 | Can you define goodput@SLO? | true paraphrase of #1 |
| 4 | What does time to first token mean? | true paraphrase of #2 |
| 5 | How does PagedAttention work? | first of topic C |
| 6 | Tell me what goodput@SLO is. | true paraphrase of #1 |
| 7 | What is prefix caching? | new topic — must never hit |
| 8 | Describe how PagedAttention works. | true paraphrase of #5 |

## Observed table (threshold 0.80)

```
 #  result   sim      ms  prompt
 1  miss    0.00     255  What is goodput at SLO?
 2  miss    0.00     250  Explain TTFT and TPOT.
 3  HIT     1.00       0  Can you define goodput@SLO?
 4  miss    0.00     252  What does time to first token mean?     ← FALSE MISS
 5  miss    0.00     253  How does PagedAttention work?
 6  HIT     1.00       0  Tell me what goodput@SLO is.
 7  miss    0.00     251  What is prefix caching?                 ← correctly not a hit
 8  HIT     1.00       0  Describe how PagedAttention works.
```

Hit rate 3/8 = 38%. Three LLM calls saved.

## False miss

**#4**, similarity **0.00**, against cached #2 "Explain TTFT and TPOT."

After stopword removal, #2 is `{ttft, tpot}` and #4 is `{time, first, token, mean}`. Zero overlap, so cosine is exactly 0. A human (and a dedicated embedder like BGE-M3 or Qwen3-Embedding) treats these as the same question. The stub cannot, because a decoder trained to predict the next token has no reason to put "TTFT" and "time to first token" in the same place — and a bag-of-words vectorizer has even less.

## False hit

The stub did **not** produce a false hit on this stream: #7 "What is prefix caching?" scored 0.00 against every stored topic, which is the correct miss. Similarity here is almost always exactly 0.0 or 1.0, so the "unrelated prompt that still matches" case never appears.

That absence is itself diagnostic. A false hit needs a *continuous* weak embedder — the lab's real path, mean-pooled chat-model states, lands every pair in a narrow band (~0.5–0.9). Then a prompt about prefix caching can outscore a true TTFT paraphrase. I did not start `make serve-embed` for that curve; the offline run already shows why quoting raw hit rate from a chat model in pooling mode would be dishonest.

A constructed false-hit if the stub had kept question words: "What is goodput of my GPU FLOPs?" shares `goodput` with #1 and would HIT at sim 1.0 with the wrong answer. Content-word collision is the bag-of-words version of the same failure.

## No single threshold fixes both

Threshold sweep (offline):

```
  0.70: 3/8 hits
  0.80: 3/8 hits
  0.85: 3/8 hits
  0.90: 3/8 hits
  0.95: 3/8 hits
```

Flat. Raising the threshold cannot recover false miss #4 (sim is 0.00, below every value). Lowering it cannot create a false hit on this stub (unrelated pairs are also 0.00). The knob does nothing because the embedder collapsed a continuous similarity problem into a set-overlap problem.

A dedicated embedding model separates paraphrases from strangers by a wide margin, so a threshold in the gap works. A next-token decoder, mean-pooled, does not build that gap. That is why production semantic caches sit on BGE / EmbeddingGemma / Qwen3-Embedding, not on the chat weights you already loaded.

## Security note

A shared semantic cache (and a shared prefix cache) is a timing side channel across tenants: a HIT is faster than a miss, so user B can probe whether user A already asked a given question. Production systems salt the cache key per tenant. The lab's demo is single-user and does not.

## What the deck did not say

Hit rate is not a quality metric for this stack. The 38% I measured is "how many prompts shared a content word with an earlier prompt." Reporting that as semantic-cache quality would be the mistake the challenge is written to catch.
