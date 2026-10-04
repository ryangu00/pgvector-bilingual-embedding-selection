# Results

All tables reproduced verbatim from the source records, complete, with a one-line measurement condition above each. Dates where the source records them.

## Table 1 — four-model comparison

Condition: retrieval quality score on our own query set; metric definition, query count and run count were not recorded. The note we kept describes the table as a separation margin ('larger gap = stronger discrimination'), not as recall; the exact formula is still not recorded. Eval set is our private 11-category eval bank (questions not published). Arrows denote query language → document language.

| Model | Dim | Index | Chinese | English | EN→CN | CN→EN |
|---|---|---|---|---|---|---|
| qwen3-0.6b | 1024 | ✅ HNSW | 0.265 | 0.526 | 0.419 | 0.496 |
| qwen3-8b | 4096 | ❌ exact scan | 0.362 | 0.590 | 0.488 | 0.463 |
| bge-m3 | 1024 | ✅ HNSW | 0.196 | 0.446 | 0.334 | 0.339 |
| nomic | 768 | ✅ HNSW | Chinese queries matched unrelated pages; no numeric score recorded | — | — | — |

## Table 2 — migration cost and post-migration verification

Condition: 2026-05-30 migration, single run. The re-embed row covers a 701-page stale subset, not a full pass over the ~1,167-page store; its per-second rate (~25.3 chunks/s, bulk embedding) is not comparable with the single-probe 106 ms latency below.

| Measure | Value |
|---|---|
| Re-embed via `embed --stale` | 1619 chunks / 701 pages / ~64 s |
| History-only pages merged from the pglite instance | 518 |
| Postgres page count before → after merge | 1167 → ~1685 |
| `doctor` integrity check | all green: 100% coverage, qwen3 embed 106 ms, width consistency OK |
| Raw pgvector retrieval probe (Chinese query about a corrupted database) | the expected page ranked first with cosine similarity 0.58–0.65 on that query |

## Table 3 — large-input acceptance battery around the `num_batch` fix

Condition: character length of embed input; run on all three nodes (2026-06-23).

| Input | Before fix | After fix |
|---|---|---|
| 6000-char document | ❌ EOF (reproduces the crash) | ✅ |
| 12000-char document | — | ✅ (token count after the fix not recorded) |
| 11500-char input | — | ✅ |
| 13000-char CJK input | — | ✅ (all three nodes) |
| Capture of a unique slug with large CJK content | — | 3/3 `status=created_or_updated` |

## Table 4 — crash mechanism and version attribution

Condition: ollama runner log evidence.

| Item | Value |
|---|---|
| Runner log at failure | `task.n_tokens=2273` → `cached n_tokens=2048` → HTTP 400 |
| Ingestion-side amplifier | the tag/assign code slices document text to 8000 characters before embedding, so even capped inputs crashed |
| Regression scope | ollama **0.30.x** regression; observed on the Mac node running 0.30.10; two other nodes on 0.24.0 / 0.23.4 did not reproduce it; not isolated further |
| Not affected | the two Dell Pro Max with GB10 nodes (0.24.0 and 0.23.4), tested on the large-input battery |

## Table 5 — native evaluation of the keyword leg

Condition: 2026-07-03, measured once. Same database as the README's Postgres + pgvector knowledge base after it had grown. 120 synthetic queries over real pages, one gold page per query, 100 of the 120 queries contain Chinese characters, k = 10, quiet job queue, single run, English text-search configuration in effect. Compared with the original README setup, the software version differed and the stack used a fine-tuned variant of the embedding model plus a trained reranker. The README's own Table 1 uses a different metric and is not affected.

| Configuration | P@10 | R@10 | MRR | nDCG@10 |
|---|---|---|---|---|
| hybrid (RRF k=60) | 0.09 | 0.93 (inflated, see metric pitfall in README Update 2026-10) | 0.74 | 0.80 |
| vector only | 0.12 | 1.21 (above 1, see metric pitfall in README Update 2026-10) | 0.76 | 0.96 |
| keyword only | 0.00 | 0.03 | 0.03 | 0.03 |

RRF constant 30, 45, 60 and 90 gave identical hybrid scores.
