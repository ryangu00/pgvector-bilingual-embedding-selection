# Pitfalls

Each pitfall is given as **symptom → root cause → fix**, expanded with **how we found it**. In the order they bear on this cookbook.

## 1. Long-chunk embed fails with 400/EOF; short inputs pass

- **Symptom.** Any embed input exceeding the ollama runner's single-batch capacity returns HTTP 400 / EOF, while short inputs succeed. The asymmetric symptom looks like a config or network problem.
- **Root cause.** The ollama runner's single-batch limit is too low by default; this was an ollama **0.30.x** regression observed on the Mac node running 0.30.10; two other nodes on 0.24.0 / 0.23.4 did not reproduce it, and the cause was not isolated further. The runner log shows `task.n_tokens=2273` → `cached n_tokens=2048` → HTTP 400. An ingestion-side amplifier made it worse: the tag/assign code slices document text to 8000 characters before embedding, so even capped inputs still crashed.
- **Fix.** Set `PARAMETER num_batch 16384` on the embedding model and retest with a large input. On 2026-06-23 this was applied to the embedding model on all three nodes and the large-input acceptance battery then passed (Table 3): 6000-char document ✅ (previously ❌ EOF), 12000-char ✅, 11500-char ✅, 13000-char CJK ✅ on all three nodes, capture of a unique slug with large CJK content 3/3 `status=created_or_updated`.
- **How we found it.** The two Dell Pro Max with GB10 nodes (ollama 0.24.0 and 0.23.4) passed the same large-input battery, so the symptom appeared only on the Mac node running 0.30.10; it was not isolated further. The runner log's token-vs-batch counters (`task.n_tokens` vs `cached n_tokens`) pointed at the batch limit rather than at the network — the rotating ephemeral ports in the failure log are the runner subprocess signature, not enough on their own to rule a network fault.

## 2. `ALTER` on the vector column errors

- **Symptom.** A direct `ALTER` on the populated pgvector vector column fails.
- **Root cause.** Existing vectors are not compatible with the target dimension on a populated column.
- **Fix.** NULL the embedding column first (which empties the old vectors), then `ALTER` the column type; re-embed the content afterward.
- **How we found it.** Hit while migrating the vector column from the nomic schema to `qwen3-embedding:0.6b`; NULL-then-ALTER cleared it.

## 3. A recipe/config edit appears to have no effect

- **Symptom.** An edit to a recipe or config file is applied, but behavior is unchanged.
- **Root cause.** The patch silently did not apply.
- **Fix.** grep the file to confirm every edit landed before retesting.
- **How we found it.** Recurring failure mode across the migration; treated as a standard verification step after any config edit.

## 4. Old embeddings keep being served after switching models

- **Symptom.** After changing the embedding model, retrieval still behaves as if the old model is in use.
- **Root cause.** The process caching the embedding configuration reads the model env only at startup.
- **Fix.** Restart whatever long-running process caches the embedding configuration whenever the embedding model changes.
- **How we found it.** Observed during the migration from nomic to qwen3; restarting that process refreshed the model.

## 5. `export <path>` writes to the wrong directory

- **Symptom.** `export <path>` puts content somewhere other than the given path.
- **Root cause.** The command ignores its path argument.
- **Fix.** Content always lands in `./export`; account for that rather than trusting the argument.
- **How we found it.** Files went to `./export` regardless of the path passed in.

## 6. Retrieval silently degrades after a tool upgrade

- **Symptom.** After a tool upgrade, retrieval quality silently degrades and raw CLI captures break.
- **Root cause.** A 2026-06-08 tool upgrade silently reset the embedding-model config entry back to `nomic-embed-text` while the dimensions config entry kept the qwen3 value — the self-contradictory pair broke every raw capture.
- **Fix.** After every upgrade, re-check both embedding config entries (model and dimensions); if they disagree, restore the correct model and dimensions, restart whatever long-running process caches the embedding configuration, check for mixed-model vectors from the inconsistent window, re-embed any content captured under the wrong model, and re-run any capture that previously failed. Add a guard script that asserts the pair is consistent.
- **How we found it.** Captures started failing after the upgrade; comparing the two config keys showed one had been reset and the other had not.

## 7. Chinese keyword search returns nothing

- **Symptom.** Full-text keyword search returns empty results for Chinese text.
- **Root cause.** Plain Postgres FTS did not return results for the Chinese queries we ran; its default text-search configuration does not segment CJK text into usable terms.
- **Fix.** The default (English) text-search configuration does not segment Chinese; it treats a whole sentence as one token. We tried pgroonga 4.0.6 with space-split `&@|` queries, and it returned relevant pages on four manual queries (2026-05-30). We dropped its index on 2026-06-06 after intermittent insert failures on large Chinese documents; the rollback record lists Postgres 17.10, pgroonga 4.0.6 and groonga 16.0.5, with no crash-safe WAL configured. If you try it, wire the `&@` query operator into your search layer, enable pgroonga's crash-safe module (`pgroonga_crash_safer` in `shared_preload_libraries`), and measure retrieval quality first; these three conditions come from our post-mortem and are untested.
- **How we found it.** Chinese queries returned empty under plain Postgres FTS; pgroonga with space-split `&@|` queries returned results. Result was four manual queries on 2026-05-30; the index was dropped on 2026-06-06 (insert failures); no recall measurement exists for it.

## 8. Embed failure logs show rotating ephemeral ports

- **Symptom.** Failure logs around large-input embed show rotating ephemeral ports, suggesting a network or config fault.
- **Root cause.** Rotating ephemeral ports are the ollama runner subprocess signature; on their own they are not enough to rule a network or config fault in or out.
- **Fix.** Check token-vs-batch counters in the runner log first (e.g. `task.n_tokens` vs `cached n_tokens`) before investigating the network.
- **How we found it.** The ports looked like a network issue until the runner log's batch counters were read, which pointed at the single-batch limit (see pitfall 1).

## 9. Automated battery reports failures that pass manually

- **Symptom.** The automated acceptance battery reports failures (e.g. 0/3) for cases that pass when run manually.
- **Root cause.** Harness truncation bug: `tail -3` ate the `captured:` header line.
- **Fix.** Stop truncating the output with `tail -3`; parse the full or structured output instead, and re-read the test slug to verify the write actually landed. `status=created_or_updated` on its own is not a complete acceptance check.
- **How we found it.** A capture of a unique slug with large CJK content was reported as 0/3 by the battery; manual reruns succeeded, and inspecting the harness showed `tail -3` had truncated the header. After the fix the same case reported 3/3 `status=created_or_updated`.

## 10. Rows missing in one feature after migration

- **Symptom.** After migration, one feature's rows are missing.
- **Root cause.** The original migration enumerated tables by hand and moved only `content_chunks` and `pages`; the `facts` table was missed.
- **Fix.** Diff the full table list before and after migration, and reconcile per-table row counts and primary keys, not just table names — a table can be present but empty, and a row-count match can still miss required associations.
- **How we found it.** A post-migration feature had no rows; diffing the table list revealed `facts` had never been migrated.
