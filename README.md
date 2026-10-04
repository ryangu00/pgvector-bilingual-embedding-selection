![banner](docs/assets/banner.png)

# Bilingual embeddings for a pgvector knowledge base — four candidates, the HNSW limit, and the `num_batch` hazard

> A measured selection of a Chinese/English text-embedding model for our personal knowledge base (~1,700 pages) on Postgres + pgvector, run on one Mac (Apple silicon, local ollama) and two Dell Pro Max with GB10 nodes (head node + worker node, TP2 over RoCE). Four open candidates were scored on Chinese, English, and both cross-lingual directions. `qwen3-embedding:0.6b` won on the combination of Chinese quality and fitting pgvector's HNSW dimension ceiling; `qwen3-8b` scored higher but was disqualified because its vectors exceed the HNSW limit and fall back to exact scan. The actual hidden hazard was not the model choice but an ollama single-batch regression that silently failed large-chunk embeddings until `PARAMETER num_batch 16384` was set. Every number below comes from the source records, with its condition; nothing here is invented.

## Why this matters

A retrieval model that "looks healthy" on short queries can be blind on the exact inputs that matter — long chunks and Chinese text. Two failures here were silent: `nomic` returned a confidently-wrong Chinese match separated from the right one by only 0.033 (indistinguishable from noise), and the ollama `0.30.x` runner returned HTTP 400/EOF on large inputs while short inputs looked fine. Both would have passed a shallow smoke test. The lesson is to assert retrieval quality on real bilingual long-chunk content, not on a held-out toy query, and to treat any asymmetric "short works / long fails" symptom as a batch-limit regression before reaching for config or network explanations.

## Update (2026-10): what happened to the Chinese keyword leg, and a metric pitfall

This section qualifies the pgroonga step in the procedure below and flags a metric bug that inflates recall.

**pgroonga trial.** On 2026-05-30 we installed pgroonga 4.0.6 on Postgres 17, built a pgroonga index on the chunk text, and routed Chinese queries to its OR operator (`&@|`) with the query split on spaces. Four manual Chinese test queries each retrieved a relevant page. No recall, MRR or latency figure was recorded.

**Insert failures and rollback.** On 2026-06-06 writes of Chinese-heavy documents intermittently failed with `pgroonga: [insert] failed to set column value: [ii][buffer][put] loop is found`. Versions: Postgres 17.10, pgroonga 4.0.6, groonga 16.0.5 (the newest available from the package manager at that time, so upgrading was ruled out). Reindexing, restarts and parameter changes were treated as symptom relief. The post-mortem found the index had zero scans (`idx_scan = 0`) and the active search code contained no pgroonga query operator. The fix was `DROP INDEX`. After the drop, capture of Chinese documents larger than 20 KB succeeded without manual reindex; twelve of twelve captures passed. The cause is inferred from the error prefix and from the failures stopping after the drop; there was no before/after control run.

**Known-good state.** With the English text-search configuration, `to_tsvector('english', <Chinese sentence>)` yields the whole sentence as a single token, so keyword retrieval on Chinese is close to dead. On 2026-07-02 both zhparser and pgroonga were confirmed still installable on the same Postgres 17 instance. A guard script that kept re-applying a Chinese configuration was removed on 2026-07-03; the English configuration stayed.

### Table 5 — native evaluation of the keyword leg

Condition: 2026-07-03, measured once. Same database as the README's Postgres + pgvector knowledge base after it had grown. 120 synthetic queries over real pages, one gold page per query, 100 of the 120 queries contain Chinese characters, k = 10, quiet job queue, single run, English text-search configuration in effect. Compared with the original README setup, the software version differed and the stack used a fine-tuned variant of the embedding model plus a trained reranker. The README's own Table 1 uses a different metric and is not affected.

| Configuration | P@10 | R@10 | MRR | nDCG@10 |
|---|---|---|---|---|
| hybrid (RRF k=60) | 0.09 | 0.93 (inflated, see below) | 0.74 | 0.80 |
| vector only | 0.12 | 1.21 (above 1, see below) | 0.76 | 0.96 |
| keyword only | 0.00 | 0.03 | 0.03 | 0.03 |

RRF constant 30, 45, 60 and 90 gave identical hybrid scores. The vector-only nDCG advantage is a property of this eval set (single-page semantic queries); we did not drop the keyword leg because production also receives exact-match queries (symbols, names, IDs). This measurement says nothing about pgroonga — pgroonga was not in the query path at that time.

**Metric pitfall: recall above 1.** Mean R@10 on the 120 queries went from 0.93 to 0.87 after a merge. A per-query comparison run on both code trees against the same database back to back at the same time showed the set of queries with a hit was identical: 102 of 120 (0.85), with no query changing from hit to miss or miss to hit. The 0.93 was inflated: the old code did not deduplicate results by page, so two chunks of the same gold page occupied two top-10 slots while only one page was relevant, giving recall = 2/1 = 2.00 on some queries. The number of queries with R@10 = 2.00 fell from ten before the merge to two after; the 8 that normalised account for about 8/120 = 0.067, the whole apparent drop. The true baseline on this set is 0.85 hit rate at k = 10. Lessons: (a) the moment recall exceeds 1, audit deduplication by page first; (b) acceptance comparisons must run both versions on the same database back to back at the same time, not today's code against last week's scoreboard; (c) a mechanism hypothesis needs per-query evidence.

**Unresolved.** Early manual-test results and later operating records are inconsistent. The records do not establish whether the pgroonga query route actually served Chinese queries between 2026-05-30 and 2026-06-06, or when it may have stopped doing so. No retrieval-quality measurement (recall, MRR, nDCG) exists for pgroonga on this database. The insert-failure cause is inferred, not isolated; whether the crash-safe module or the MeCab tokenizer would prevent it is untested. The Chinese tokenizer A/B (zhparser vs English) came from a harness we later disowned; a clean re-run on the native evaluation path was not done. The exact formula of Table 1's scores is not recorded.

## Hardware and stack

| | |
|---|---|
| Hardware | 1 Mac (Apple silicon, local ollama) + 2 Dell Pro Max with GB10 (head node + worker node) |
| Store | Postgres with pgvector **0.8.2** — stores vectors up to 16,000 dims, but builds an HNSW index only up to 2,000 dims; a 4,096-dim model therefore runs as an exact scan on this schema (usable, but not indexed) |
| Chosen model | `qwen3-embedding:0.6b` served by ollama |
| Model flag | Modelfile `PARAMETER num_batch 16384` (ollama default is much lower — see crash log below) |
| ollama versions | Mac: 0.30.10 (affected by the regression); Dell Pro Max with GB10 nodes: 0.24.0 and 0.23.4 (tested, not affected) |
| Chinese keyword search | pgroonga 4.0.6 was tried on 2026-05-30 and its index was dropped on 2026-06-06; not part of our later known-good configuration (see Update 2026-10) |

The candidates benchmarked are public upstream models: `qwen3-embedding:0.6b`, `qwen3-8b`, `bge-m3`, and `nomic-embed-text` (served via ollama). Our retrieval-quality numbers come from our private 11-category eval bank (questions not published); retrieval quality score on our own query set; metric definition, query count and run count were not recorded, so the scores are reproduced as recorded rather than re-derived. The note we kept describes the table as a separation margin ('larger gap = stronger discrimination'), not as recall; the exact formula is still not recorded.

## Procedure as run

The source records the procedure as a sequence of decisions and verifications, not a single runnable script. The migration and evaluation scripts are not published; where a launch command is not recorded, this cookbook says so rather than inventing one.

1. **Score candidates on bilingual retrieval.** Score the four candidates on Chinese, English, and cross-lingual EN->CN and CN->EN retrieval of the knowledge base (Table 1). The eval set is our private 11-category eval bank (questions not published); the metric definition, query count, and run count are not recorded in the source. The note we kept describes the table as a separation margin ('larger gap = stronger discrimination'), not as recall; the exact formula is still not recorded.
2. **Screen against pgvector constraints.** Any candidate wider than the HNSW index limit (2,000 dims) gets no ANN index and falls back to exact scan. This disqualified `qwen3-8b` (4,096 dims) despite its higher scores.
3. **Choose `qwen3-embedding:0.6b` and migrate the vector column.** Migrate from the nomic schema by NULL-ing the embedding column first, then `ALTER` the column type — a direct `ALTER` on a populated pgvector column fails.
4. **Re-embed existing content.** Re-embed stale content with the `embed --stale` command of the knowledge-base tool (cost in Table 2).
5. **Merge the history-only instance.** Merge a second embedded-database (pglite) instance holding history-only pages; the Postgres page count grows accordingly (Table 2).
6. **Restart the caching process.** Restart whatever long-running process caches the embedding configuration after the embedding-model change — that process reads the model env only at startup.
7. **Verify.** Run the `doctor` integrity check plus a raw pgvector retrieval probe (Table 2).
8. **Apply the batch fix preventively (2026-06-23).** Apply `PARAMETER num_batch 16384` to the embedding model on all three nodes and run the large-input acceptance battery (Tables 3–4).
9. **Optional experiment, rolled back: pgroonga for Chinese keyword search.** Plain Postgres FTS with the English configuration returns nothing useful on Chinese. On 2026-05-30 we installed pgroonga 4.0.6, built an index on the chunk text and routed Chinese queries through its OR operator (`&@|`), splitting the query on spaces; four manual Chinese test queries each returned a relevant page. We never measured retrieval quality with it, and we dropped the index on 2026-06-06 after intermittent insert failures (see Update 2026-10). Do not treat this step as a verified recommendation.

> The exact ollama serve command and Modelfile build commands are not recorded in the source; the only recorded flag is `PARAMETER num_batch 16384`. Do not assume a specific port or launch invocation.

## Results

All tables are reproduced verbatim from the source with the measurement condition noted above each. Full tables live in `docs/results.md`.

### Table 1 — four-model comparison

Condition: retrieval quality score on our own query set; metric definition, query count and run count were not recorded. The note we kept describes the table as a separation margin ('larger gap = stronger discrimination'), not as recall; the exact formula is still not recorded. Arrows denote query language → document language.

| Model | Dim | Index | Chinese | English | EN→CN | CN→EN |
|---|---|---|---|---|---|---|
| qwen3-0.6b | 1024 | ✅ HNSW | 0.265 | 0.526 | 0.419 | 0.496 |
| qwen3-8b | 4096 | ❌ exact scan | 0.362 | 0.590 | 0.488 | 0.463 |
| bge-m3 | 1024 | ✅ HNSW | 0.196 | 0.446 | 0.334 | 0.339 |
| nomic | 768 | ✅ HNSW | Chinese queries matched unrelated pages; no numeric score recorded | — | — | — |

### Table 2 — migration cost and verification

Condition: 2026-05-30 migration, single run.

| Measure | Value |
|---|---|
| Re-embed via `embed --stale` | 1619 chunks / 701 pages / ~64 s |
| History-only pages merged from the pglite instance | 518 |
| Postgres page count before → after merge | 1167 → ~1685 |
| `doctor` integrity check | all green: 100% coverage, qwen3 embed 106 ms, width consistency OK |
| Raw pgvector retrieval probe (Chinese query about a corrupted database) | the expected page ranked first with cosine similarity 0.58–0.65 on that query |

### Table 3 — large-input acceptance battery around the `num_batch` fix

Condition: character length of embed input; run on all three nodes (2026-06-23).

| Input | Before fix | After fix |
|---|---|---|
| 6000-char document | ❌ EOF (reproduces the crash) | ✅ |
| 12000-char document | — | ✅ (token count after the fix not recorded) |
| 11500-char input | — | ✅ |
| 13000-char CJK input | — | ✅ (all three nodes) |
| Capture of a unique slug with large CJK content | — | 3/3 `status=created_or_updated` |

### Table 4 — crash mechanism and version attribution

Condition: ollama runner log evidence.

| Item | Value |
|---|---|
| Runner log at failure | `task.n_tokens=2273` → `cached n_tokens=2048` → HTTP 400 |
| Ingestion-side amplifier | the tag/assign code slices document text to 8000 characters before embedding, so even capped inputs crashed |
| Regression scope | ollama **0.30.x** regression; observed on the Mac node running 0.30.10; two other nodes on 0.24.0 / 0.23.4 did not reproduce it; not isolated further |
| Not affected | the two Dell Pro Max with GB10 nodes (0.24.0 and 0.23.4), tested on the large-input battery |

## What did not work

- **nomic on Chinese.** A Chinese question about RAM exhaustion / capacity expansion on a GB10-class server matched an unrelated page on a different topic; a 0.033 similarity gap between the top hits on one query, which we read as no real separation. Chinese retrieval with nomic was effectively blind on that query.
- **qwen3-8b.** Highest Chinese and English scores in Table 1, but its vector width exceeds the pgvector HNSW limit for this schema, so it runs as an exact scan (usable, but not indexed under this fixed dimension and index scheme).
- **Large-chunk embedding (pre-fix).** Any input exceeding the ollama runner's single-batch capacity failed with HTTP 400/EOF while short inputs looked healthy — the asymmetric symptom made it look like a config or network problem.
- **False test failures from the harness.** `tail -3` truncated the "captured:" header and the battery falsely reported 0/3.
- **2026-06-08 regression.** A tool upgrade silently reset the embedding-model config entry back to `nomic-embed-text` while the dimensions config entry kept the qwen3 value — the self-contradictory pair broke every raw CLI capture.
- **Incomplete table migration.** The original migration moved only `content_chunks` and `pages`; the `facts` table was missed.
- **Postgres FTS on Chinese.** Full-text keyword search with the English configuration returned empty or useless results for Chinese text. pgroonga made four manual test queries return relevant pages (2026-05-30), but the index was dropped a week later after intermittent insert failures and we have no retrieval-quality measurement for it. A later native evaluation with the English configuration in place scored 0.03 recall@10 for the keyword leg alone (see Update 2026-10).

## Pitfalls

Symptom → root cause → fix, in the order they bear on this cookbook. Expanded with "how we found it" in `docs/pitfalls.md`.

- Long-chunk embed fails with 400/EOF but short inputs pass → ollama runner single-batch limit (0.30.x regression; observed on the Mac node running 0.30.10; two other nodes on 0.24.0 / 0.23.4 did not reproduce it; not isolated further) → set `PARAMETER num_batch 16384` on the embedding model and retest with a large input.
- `ALTER` on the vector column errors → existing vectors are not compatible with the target dimension on a populated column → NULL the embedding column first, then ALTER.
- A recipe/config edit appears to have no effect → the patch silently did not apply → grep the file to confirm every edit landed.
- Old embeddings keep being served after switching models → the process caching the embedding configuration reads the model env only at startup → restart whatever long-running process caches the embedding configuration whenever the embedding model changes.
- `export <path>` writes to the wrong directory → the command ignores its path argument → content always lands in `./export`.
- Retrieval silently degrades after a tool upgrade → the upgrade reset one embedding config key but not its paired dimension key → re-check both embedding config entries after every upgrade and add a guard script.
- Chinese keyword search returns nothing → the default (English) text-search configuration does not segment Chinese; it treats a whole sentence as one token → we tried pgroonga with space-split `&@|` queries and it returned relevant pages on four manual queries, but we rolled it back after intermittent insert failures on large Chinese documents (pgroonga 4.0.6, groonga 16.0.5, no crash-safe WAL configured). If you try it, wire the query operator into your search layer, enable pgroonga's crash-safe module, and measure retrieval quality first; these three conditions come from our post-mortem and are untested.
- Embed failure logs show rotating ephemeral ports → that is the ollama runner subprocess signature, not enough on its own to rule a network or config fault → check token-vs-batch counters in the runner log first.
- Automated battery reports failures that pass manually → harness truncation bug (`tail -3` ate the header line) → stop truncating the output with `tail -3`, parse the full or structured output, and re-read the test slug to verify the write.
- Rows missing in one feature after migration → migration enumerated tables by hand and missed `facts` → diff the full table list before and after, and reconcile per-table row counts and primary keys, not just table names.

## Files

- `README.md` — this cookbook.
- `docs/results.md` — all tables, complete, with measurement conditions.
- `docs/pitfalls.md` — pitfalls expanded with symptom / root cause / fix / how we found it.
- `docs/make_banner.py` — banner generator (pure PIL, house style); produces `docs/assets/banner.png`.
- `docs/assets/banner.png` — generated banner (run `docs/make_banner.py`).

## License

Apache-2.0.
