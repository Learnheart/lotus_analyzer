# B0 — Operator Inventory

> **Tổng quan** tất cả các semantic operator trong LOTUS. Bảng này là master reference cho toàn bộ hệ thống operator.

## Master Table

| Operator | Method | File | Lines | Type | Input | Output | Description |
|---|---|---|---|---|---|---|---|
| `sem_filter` | `SemFilterDataframe.__call__` | `sem_ops/sem_filter.py` | 24-129 (core), 225-593 (accessor) | Unary | DataFrame + langex predicate | Filtered DataFrame (boolean per row) | Lọc từng row theo điều kiện ngôn ngữ tự nhiên, trả về True/False |
| `sem_map` | `SemMapDataframe.__call__` | `sem_ops/sem_map.py` | 14-118 (core), 121-279 (accessor) | Unary | DataFrame + langex instruction | DataFrame + new column (string per row) | Chuyển đổi mỗi row thành giá trị mới qua LLM |
| `sem_join` | `SemJoinDataframe.__call__` | `sem_ops/sem_join.py` | 16-177 (core), 606-822 (accessor) | Binary | 2 DataFrames + join predicate | Joined DataFrame | Join 2 bảng theo điều kiện ngữ nghĩa, M x N comparisons |
| `sem_agg` | `SemAggDataframe.__call__` | `sem_ops/sem_agg.py` | 60-223 (core), 226-441 (accessor) | Unary (many-to-one) | DataFrame + aggregation instruction | Single-row DataFrame | Tổng hợp nhiều row thành 1 kết quả, dùng hierarchical tree |
| `sem_topk` | `SemTopKDataframe.__call__` | `sem_ops/sem_topk.py` | 624-847 (accessor) | Unary | DataFrame + ranking instruction + K | Top-K DataFrame (sorted) | Sắp xếp và trả về K row tốt nhất theo tiêu chí ngữ nghĩa |
| `sem_extract` | `SemExtractDataFrame.__call__` | `sem_ops/sem_extract.py` | 15-108 (core), 111-256 (accessor) | Unary | DataFrame + input_cols + output_cols schema | DataFrame + extracted columns | Trích xuất thông tin có cấu trúc từ text, trả về JSON |
| `sem_search` | `SemSearchDataframe.__call__` | `sem_ops/sem_search.py` | 10-157 (accessor) | Unary | DataFrame + query + K | Filtered DataFrame | Tìm kiếm vector, không dùng LLM. Có thể rerank |
| `sem_sim_join` | `SemSimJoinDataframe.__call__` | `sem_ops/sem_sim_join.py` | 12-166 (accessor) | Binary | 2 DataFrames + left_on + right_on + K | Joined DataFrame with scores | Join bằng embedding similarity, dùng vector index KNN |
| `sem_dedup` | `SemDedupByDataframe.__call__` | `sem_ops/sem_dedup.py` | 10-91 (accessor) | Unary | DataFrame + col_name + threshold | Deduplicated DataFrame | Loại bỏ duplicate dùng connected components trên similarity graph |
| `sem_index` | `SemIndexDataframe.__call__` | `sem_ops/sem_index.py` | 9-77 (accessor) | Unary (side-effect) | DataFrame + col_name + index_dir | DataFrame (unchanged, but index created) | Tạo vector index cho cột, cần thiết cho search/join/cluster |
| `sem_partition_by` | `SemPartitionByDataframe.__call__` | `sem_ops/sem_partition_by.py` | 8-67 (accessor) | Unary | DataFrame + partition_fn | DataFrame + `_lotus_partition_id` column | Phân nhóm DataFrame theo hàm partition (vd: cluster) |
| `sem_cluster_by` | `SemClusterByDataframe.__call__` | `sem_ops/sem_cluster_by.py` | 10-86 (accessor) | Unary | DataFrame + col_name + ncentroids | DataFrame + `cluster_id` column | Phân cụm dùng FAISS kmeans trên vector embeddings |

## Phân loại theo dependency

### Operators yêu cầu LLM (`lotus.settings.lm`)
- `sem_filter` (sem_filter.py:351)
- `sem_map` (sem_map.py:229)
- `sem_join` (sem_join.py:685-689)
- `sem_agg` (sem_agg.py:364-367)
- `sem_topk` (sem_topk.py:747-751)
- `sem_extract` (sem_extract.py:216-219)

### Operators yêu cầu Retrieval Model (`lotus.settings.rm`) + Vector Store (`lotus.settings.vs`)
- `sem_search` (sem_search.py:104-109)
- `sem_sim_join` (sem_sim_join.py:101-106)
- `sem_dedup` (sem_dedup.py:38-43)
- `sem_index` (sem_index.py:67-72)
- `sem_cluster_by` (sem_cluster_by.py:67-72)

### Operators không yêu cầu model
- `sem_partition_by` (sem_partition_by.py:60-67) — chỉ cần một partition function

## Phân loại theo có dùng LLM call

| Category | Operators | Chi phí |
|---|---|---|
| LLM-heavy | sem_filter, sem_map, sem_extract | O(N) LLM calls |
| LLM-heavy (pairwise) | sem_join, sem_topk | O(N*M) hoặc O(N log N) LLM calls |
| LLM-heavy (hierarchical) | sem_agg | O(N / batch_size * log(levels)) |
| Embedding-only | sem_search, sem_sim_join, sem_dedup, sem_index, sem_cluster_by | O(N) embedding calls, 0 LLM calls |
| No model | sem_partition_by | 0 calls |

## Registration Pattern chung

Tất cả operators đều dùng cùng một pattern:
- Decorator: `@pd.api.extensions.register_dataframe_accessor("sem_xxx")` (vd: sem_filter.py:225)
- Cache: `@operator_cache` decorator trên `__call__` (vd: sem_filter.py:333)
- Validation: `_validate()` kiểm tra input là DataFrame (vd: sem_filter.py:318-331)
- Langex parsing: `lotus.nl_expression.parse_cols()` trích xuất column names (vd: sem_filter.py:358)
- Data serialization: `task_instructions.df2multimodal_info()` chuyển DataFrame sang multimodal format (vd: sem_filter.py:367)
