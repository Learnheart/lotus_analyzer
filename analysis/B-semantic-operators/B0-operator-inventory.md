# B0 — Operator Inventory

> **Tong quan** tat ca cac semantic operator trong LOTUS. Bang nay la master reference cho toan bo he thong operator.

## Master Table

| Operator | Method | File | Lines | Type | Input | Output | Description |
|---|---|---|---|---|---|---|---|
| `sem_filter` | `SemFilterDataframe.__call__` | `sem_ops/sem_filter.py` | 24-129 (core), 225-593 (accessor) | Unary | DataFrame + langex predicate | Filtered DataFrame (boolean per row) | Loc tung row theo dieu kien ngon ngu tu nhien, tra ve True/False |
| `sem_map` | `SemMapDataframe.__call__` | `sem_ops/sem_map.py` | 14-118 (core), 121-279 (accessor) | Unary | DataFrame + langex instruction | DataFrame + new column (string per row) | Chuyen doi moi row thanh gia tri moi qua LLM |
| `sem_join` | `SemJoinDataframe.__call__` | `sem_ops/sem_join.py` | 16-177 (core), 606-822 (accessor) | Binary | 2 DataFrames + join predicate | Joined DataFrame | Join 2 bang theo dieu kien ngu nghia, M x N comparisons |
| `sem_agg` | `SemAggDataframe.__call__` | `sem_ops/sem_agg.py` | 60-223 (core), 226-441 (accessor) | Unary (many-to-one) | DataFrame + aggregation instruction | Single-row DataFrame | Tong hop nhieu row thanh 1 ket qua, dung hierarchical tree |
| `sem_topk` | `SemTopKDataframe.__call__` | `sem_ops/sem_topk.py` | 624-847 (accessor) | Unary | DataFrame + ranking instruction + K | Top-K DataFrame (sorted) | Sap xep va tra ve K row tot nhat theo tieu chi ngu nghia |
| `sem_extract` | `SemExtractDataFrame.__call__` | `sem_ops/sem_extract.py` | 15-108 (core), 111-256 (accessor) | Unary | DataFrame + input_cols + output_cols schema | DataFrame + extracted columns | Trich xuat thong tin co cau truc tu text, tra ve JSON |
| `sem_search` | `SemSearchDataframe.__call__` | `sem_ops/sem_search.py` | 10-157 (accessor) | Unary | DataFrame + query + K | Filtered DataFrame | Tim kiem vector, khong dung LLM. Co the rerank |
| `sem_sim_join` | `SemSimJoinDataframe.__call__` | `sem_ops/sem_sim_join.py` | 12-166 (accessor) | Binary | 2 DataFrames + left_on + right_on + K | Joined DataFrame with scores | Join bang embedding similarity, dung vector index KNN |
| `sem_dedup` | `SemDedupByDataframe.__call__` | `sem_ops/sem_dedup.py` | 10-91 (accessor) | Unary | DataFrame + col_name + threshold | Deduplicated DataFrame | Loai bo duplicate dung connected components tren similarity graph |
| `sem_index` | `SemIndexDataframe.__call__` | `sem_ops/sem_index.py` | 9-77 (accessor) | Unary (side-effect) | DataFrame + col_name + index_dir | DataFrame (unchanged, but index created) | Tao vector index cho cot, can thiet cho search/join/cluster |
| `sem_partition_by` | `SemPartitionByDataframe.__call__` | `sem_ops/sem_partition_by.py` | 8-67 (accessor) | Unary | DataFrame + partition_fn | DataFrame + `_lotus_partition_id` column | Phan nhom DataFrame theo ham partition (vd: cluster) |
| `sem_cluster_by` | `SemClusterByDataframe.__call__` | `sem_ops/sem_cluster_by.py` | 10-86 (accessor) | Unary | DataFrame + col_name + ncentroids | DataFrame + `cluster_id` column | Phan cum dung FAISS kmeans tren vector embeddings |

## Phan loai theo dependency

### Operators yeu cau LLM (`lotus.settings.lm`)
- `sem_filter` (sem_filter.py:351)
- `sem_map` (sem_map.py:229)
- `sem_join` (sem_join.py:685-689)
- `sem_agg` (sem_agg.py:364-367)
- `sem_topk` (sem_topk.py:747-751)
- `sem_extract` (sem_extract.py:216-219)

### Operators yeu cau Retrieval Model (`lotus.settings.rm`) + Vector Store (`lotus.settings.vs`)
- `sem_search` (sem_search.py:104-109)
- `sem_sim_join` (sem_sim_join.py:101-106)
- `sem_dedup` (sem_dedup.py:38-43)
- `sem_index` (sem_index.py:67-72)
- `sem_cluster_by` (sem_cluster_by.py:67-72)

### Operators khong yeu cau model
- `sem_partition_by` (sem_partition_by.py:60-67) — chi can mot partition function

## Phan loai theo co dung LLM call

| Category | Operators | Chi phi |
|---|---|---|
| LLM-heavy | sem_filter, sem_map, sem_extract | O(N) LLM calls |
| LLM-heavy (pairwise) | sem_join, sem_topk | O(N*M) hoac O(N log N) LLM calls |
| LLM-heavy (hierarchical) | sem_agg | O(N / batch_size * log(levels)) |
| Embedding-only | sem_search, sem_sim_join, sem_dedup, sem_index, sem_cluster_by | O(N) embedding calls, 0 LLM calls |
| No model | sem_partition_by | 0 calls |

## Registration Pattern chung

Tat ca operators deu dung cung mot pattern:
- Decorator: `@pd.api.extensions.register_dataframe_accessor("sem_xxx")` (vd: sem_filter.py:225)
- Cache: `@operator_cache` decorator tren `__call__` (vd: sem_filter.py:333)
- Validation: `_validate()` kiem tra input la DataFrame (vd: sem_filter.py:318-331)
- Langex parsing: `lotus.nl_expression.parse_cols()` trich xuat column names (vd: sem_filter.py:358)
- Data serialization: `task_instructions.df2multimodal_info()` chuyen DataFrame sang multimodal format (vd: sem_filter.py:367)
