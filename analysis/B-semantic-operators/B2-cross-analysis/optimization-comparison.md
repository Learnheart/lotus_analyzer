# B2 — Optimization Comparison Matrix

> **So sanh** cac optimization techniques giua tat ca operators.

## Master Optimization Matrix

| Operator | Batching | Caching | Cascading | Early-term | Sampling | Safe Mode | Parallel GroupBy |
|---|---|---|---|---|---|---|---|
| sem_filter | Yes (sem_filter.py:112) | Yes (sem_filter.py:333) | Yes — HELPER_LM + EMBEDDING (sem_filter.py:396-462) | No | Yes — importance_sampling (sem_filter.py:443) | Yes (sem_filter.py:107) | No |
| sem_map | Yes (sem_map.py:102) | Yes (sem_map.py:214) | No | No | No | Yes (sem_map.py:96) | No |
| sem_join | Yes (sem_join.py:137) | Yes (sem_join.py:669) | Yes — SF vs MSF optimizer (sem_join.py:417-527) | No | Yes — importance_sampling (sem_join.py:566) | Partial (sem_join.py:104) | No |
| sem_agg | Yes — token-aware (sem_agg.py:183) | Yes (sem_agg.py:353) | No | No | No | No (TODO sem_agg.py:151) | Yes (sem_agg.py:398) |
| sem_topk | Yes (sem_topk.py:169) | Yes (sem_topk.py:734) | Yes — helper_lm (sem_topk.py:176-273) | Partial — quicksort top-K (sem_topk.py:479) | No | Yes (sem_topk.py:393) | Yes (sem_topk.py:772) |
| sem_extract | Yes (sem_extract.py:90) | Yes (sem_extract.py:201) | No | No | No | Yes (sem_extract.py:82) | No |
| sem_search | N/A | Yes (sem_search.py:91) | N/A | No | No | No | No |
| sem_sim_join | Yes (sem_sim_join.py:130) | Yes (sem_sim_join.py:84) | N/A | No | No | No | No |
| sem_dedup | N/A | Yes (sem_dedup.py:32) | N/A | No | No | No | No |
| sem_index | Yes (sem_index.py:74) | Yes (sem_index.py:61) | N/A | No | No | No | No |
| sem_partition_by | N/A | Yes (sem_partition_by.py:60) | N/A | No | No | No | No |
| sem_cluster_by | N/A | Yes (sem_cluster_by.py:57) | N/A | No | No | No | No |

## Batching Chi Tiet

### LLM Batching:
- **sem_filter**: Gui toan bo `inputs` list 1 lan: `model(inputs, ...)` (sem_filter.py:112-114)
- **sem_map**: Tuong tu: `model(inputs, ...)` (sem_map.py:102)
- **sem_join**: Gui toan bo M*N pairs qua sem_filter 1 lan (sem_join.py:137)
- **sem_extract**: `model(inputs, ...)` (sem_extract.py:90-92)
- **sem_topk**: `compare_batch_binary` gui batch pairs (sem_topk.py:169). Nhung HeapDoc.`__lt__` gui 1 pair/call (sem_topk.py:548) — heap method khong batch

### Token-aware Batching (sem_agg only):
- Dem tokens cua template + documents (sem_agg.py:174-181)
- Gop documents vao batch cho den khi dat limit (sem_agg.py:183)
- Limit: `model.max_ctx_len - model.max_tokens` (sem_agg.py:183)

### Embedding Batching:
- **sem_index**: `rm(df[col_name].tolist())` — batch toan bo column (sem_index.py:74)
- **sem_sim_join**: `rm.convert_query_to_query_vector(queries)` — batch queries (sem_sim_join.py:130)

## Cascade Chi Tiet

### sem_filter Cascade (sem_filter.py:396-530):

```
                    ┌─────────────┐
                    │  Input Data  │
                    └──────┬──────┘
                           │
                    ┌──────▼──────┐
                    │ Proxy Model │
                    │ (Helper LM  │
                    │  or RM+VS)  │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │            │            │
     ┌────────▼──────┐     │   ┌────────▼──────┐
     │ High Conf (+) │     │   │ High Conf (-) │
     │ score >= t+   │     │   │ score <= t-   │
     │ → Accept True │     │   │ → Accept False│
     └───────────────┘     │   └───────────────┘
                    ┌──────▼──────┐
                    │  Low Conf   │
                    │ t- < s < t+ │
                    │ → Oracle LM │
                    └─────────────┘
```

**Proxy models**:
1. `ProxyModel.HELPER_LM` (sem_filter.py:407): Chay `lotus.settings.helper_lm` voi logprobs, calibrate qua `calibrate_llm_logprobs()` (cascade_utils.py:33-39)
2. `ProxyModel.EMBEDDING_MODEL` (sem_filter.py:435): Dung `sem_search` de lay similarity scores

**Threshold learning** (sem_filter.py:132-222):
1. Importance sampling lay sample (cascade_utils.py:8-30)
2. Chay oracle LLM tren sample (sem_filter.py:196-208)
3. `learn_cascade_thresholds()` tim (t+, t-) thoa man recall/precision targets (cascade_utils.py:42-144)

### sem_join Cascade (sem_join.py:180-333, 417-527):

```
                    ┌─────────────┐
                    │  L1 x L2    │
                    └──────┬──────┘
                           │
              ┌────────────┼────────────┐
              │                         │
     ┌────────▼──────┐       ┌─────────▼────────┐
     │  SF Plan      │       │  MSF Plan        │
     │  sim_join +   │       │  map(L1) +       │
     │  threshold    │       │  sim_join +       │
     │  learn        │       │  threshold learn  │
     └────────┬──────┘       └─────────┬────────┘
              │                         │
              └──────────┬──────────────┘
                         │
                  ┌──────▼──────┐
                  │ Choose min  │
                  │ cost plan   │
                  └──────┬──────┘
                         │
              ┌──────────┼──────────┐
              │                     │
     ┌────────▼──────┐    ┌────────▼──────┐
     │ High Conf     │    │ Low Conf      │
     │ → Accept      │    │ → Oracle LM   │
     └───────────────┘    └───────────────┘
```

### sem_topk Cascade (sem_topk.py:176-273):
- Dung `helper_lm` voi logprobs cho comparisons
- `cascade_threshold`: confidence threshold (sem_topk.py:251)
- Low confidence → send to large model (sem_topk.py:256-270)

## Safe Mode Chi Tiet

| Operator | Estimation method | File:Line |
|---|---|---|
| sem_filter | `sum(model.count_tokens(input))` * N calls | sem_filter.py:108-110 |
| sem_map | `sum(model.count_tokens(input))` * N calls | sem_map.py:97-99 |
| sem_join | `tokens_per_call * M*N` | sem_join.py:106-120 |
| sem_topk | Method-specific estimates | sem_topk.py:393-399, 597-603 |
| sem_extract | `sum(model.count_tokens(input))` * N calls | sem_extract.py:83-85 |
| sem_agg | TODO — chua implement | sem_agg.py:151-153 |

**show_safe_mode()** (utils.py): Hien thi estimated cost va total calls, cho user confirm truoc khi chay.

## Parallel GroupBy

Chi 2 operators ho tro parallel group_by:
- **sem_agg**: `ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads)` (sem_agg.py:398)
- **sem_topk**: `ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads)` (sem_topk.py:772)

**Pattern**: Map `process_group()` static method qua executor.

## Cost Complexity

| Operator | LLM Calls | Embedding Calls | Total Complexity |
|---|---|---|---|
| sem_filter | O(N) | 0 (hoac O(N) cho cascade) | O(N) |
| sem_map | O(N) | 0 | O(N) |
| sem_join | O(M*N) | O(M+N) cho cascade | O(M*N) worst case |
| sem_agg | O(N/batch * levels) | 0 | O(N/B * log(N/B)) |
| sem_topk (quick) | O(N log K) avg | 0 (hoac O(N) cho quick-sem) | O(N log K) |
| sem_topk (heap) | O(N log N) | 0 | O(N log N) |
| sem_topk (naive) | O(N^2) | 0 | O(N^2) |
| sem_extract | O(N) | 0 | O(N) |
| sem_search | 0 | 1 query encode | O(K) retrieval |
| sem_sim_join | 0 | O(M) query encode | O(M*K) retrieval |
| sem_dedup | 0 | O(N) self-join | O(N^2) comparisons |
| sem_index | 0 | O(N) embeddings | O(N) |
| sem_partition_by | 0 | 0 | O(fn complexity) |
| sem_cluster_by | 0 | O(N) vector lookup | O(N*niter) kmeans |
