# B2 — Operator Composition & Pipeline

> **Phân tích** cách các operators compose thành pipelines, data flow giữa operators.

## 1. Composition Model

LOTUS operators compose qua DataFrame chaining — output của operator này là input của operator tiếp theo:

```python
result = (
    df
    .sem_index("text", "text_idx")          # Side-effect: tạo index
    .sem_search("text", "AI topic", K=20)   # Filter: 20 rows
    .sem_filter("The {text} is about ML")   # Filter: subset of 20
    .sem_map("Summarize {text}")            # Transform: thêm column _map
    .sem_agg("Combine all summaries")       # Aggregate: 1 row
)
```

## 2. Data Flow Giữa Operators

### DataFrame Attributes Flow

Mỗi DataFrame có `attrs` dict chứa metadata. Quan trọng nhất là `index_dirs`:

```
df.attrs["index_dirs"] = {
    "col_name": "path/to/index_dir"
}
```

**Operators tạo index_dirs**:
- `sem_index`: `self._obj.attrs["index_dirs"][col_name] = index_dir` (sem_index.py:76)

**Operators truyền index_dirs**:
- `sem_search`: `new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)` (sem_search.py:141)
- `sem_filter` (return_all=False): `new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)` (sem_filter.py:564)

**Operators không truyền index_dirs**: Hầu hết operators khác (sem_map, sem_extract, etc.) dùng `self._obj.copy()` — attrs có thể bị mất!

### _lotus_partition_id Flow

```
df.sem_partition_by(fn) → df["_lotus_partition_id"] = partition_ids
                           ↓
df.sem_agg(...)          → check "_lotus_partition_id" in df.columns (sem_agg.py:402)
                           → sort by partition_id (sem_agg.py:403)
                           → use as partition_ids for hierarchical agg (sem_agg.py:404)
```

### cluster_id Flow

```
df.sem_cluster_by("col", n) → df["cluster_id"] = cluster_ids
                                ↓
                              (User can use for group_by or further analysis)
```

## 3. Common Pipeline Patterns

### Pattern 1: Index → Search → Process
```python
df = df.sem_index("text", "text_idx")
results = df.sem_search("text", "query", K=10)
results = results.sem_map("Extract key info from {text}")
```

**Data flow**:
```
DataFrame(N rows) → sem_index → DataFrame(N rows, index created)
                   → sem_search → DataFrame(10 rows, index_dirs preserved)
                   → sem_map → DataFrame(10 rows + _map column)
```

### Pattern 2: Filter → Map → Aggregate
```python
result = (
    df.sem_filter("The {text} is relevant to topic X")
      .sem_map("Summarize {text} in one sentence")
      .sem_agg("Combine all summaries from {_map}")
)
```

**Data flow**:
```
DataFrame(N rows) → sem_filter → DataFrame(M rows, M <= N)
                   → sem_map → DataFrame(M rows + _map column)
                   → sem_agg → DataFrame(1 row, _output column)
```

### Pattern 3: Cluster → Partition → Aggregate
```python
result = (
    df.sem_index("text", "text_idx")
      .sem_cluster_by("text", ncentroids=3)
      .sem_partition_by(lotus.utils.cluster("text", 3))
      .sem_agg("Summarize {text}")
)
```

**Data flow**:
```
DataFrame(N rows) → sem_index → DataFrame(N, index)
                   → sem_cluster_by → DataFrame(N, +cluster_id)
                   → sem_partition_by → DataFrame(N, +_lotus_partition_id)
                   → sem_agg → DataFrame(1 row, hierarchical by partition)
```

### Pattern 4: Semantic Join
```python
result = df1.sem_join(df2, "the {article} belongs to {category}")
```

**Data flow**:
```
DataFrame1(M rows) + DataFrame2(N rows)
    → sem_join → M*N comparisons → DataFrame(K rows, K <= M*N)
```

### Pattern 5: Sim Join → Dedup
```python
df = df.sem_index("title", "title_idx")
clean = df.sem_dedup("title", threshold=0.9)
```

**Data flow**:
```
DataFrame(N rows) → sem_index → DataFrame(N, index)
                   → sem_dedup → sem_sim_join(self, self) → connected components
                   → DataFrame(M rows, M <= N)
```

### Pattern 6: TopK với Embedding Pre-sort
```python
df = df.sem_index("title", "title_idx")
top = df.sem_topk("The {title} is most relevant", K=5, method="quick-sem")
```

**Data flow**:
```
DataFrame(N rows) → sem_topk(quick-sem)
                   → sem_index + sem_search (pre-sort by embedding)
                   → llm_quicksort with embedding-optimized pivots
                   → DataFrame(5 rows, sorted)
```

## 4. Intermediate DataFrame State

Mỗi operator trả về DataFrame với specific state changes:

| Operator | Rows | New Columns | Attrs Changes | Index Changes |
|---|---|---|---|---|
| sem_index | Same | None | `index_dirs[col] = dir` | None |
| sem_search | <= K | Optional `vec_scores_sim_score` | `index_dirs` preserved | Reindexed to matched rows |
| sem_filter (default) | <= N | None | `index_dirs` preserved | Reindexed to True rows |
| sem_filter (return_all) | Same | `filter_label` | None | None |
| sem_map | Same | `_map` (or custom suffix) | None (copy) | None |
| sem_extract | Same | Output col names from schema | None (copy) | None |
| sem_join | Variable | Joined columns + optional explanation | None | New index |
| sem_sim_join | N * K | `_scores` | None | Repeated left index |
| sem_agg | 1 (or 1/group) | `_output` | None | Reset |
| sem_topk | K | Optional `explanation` | None | Reset |
| sem_dedup | <= N | None | None | Subset of original |
| sem_partition_by | Same | `_lotus_partition_id` | None | None (mutates) |
| sem_cluster_by | Same | `cluster_id` | None | None (mutates) |

## 5. Composition Risks

### Risk 1: index_dirs Loss
```python
# index_dirs bị mất sau sem_map vì dùng .copy()
df = df.sem_index("text", "idx")
df2 = df.sem_map("Process {text}")  # df2 không có index_dirs!
df2.sem_search("text", "query", K=5)  # KeyError!
```

### Risk 2: sem_index Init Overwrite
```python
df.attrs["index_dirs"] = {"col1": "idx1"}
df.sem_index  # __init__ runs, sets attrs["index_dirs"] = {} → mất "col1" index!
```

### Risk 3: Mutate vs Copy Confusion
```python
df.sem_partition_by(fn)  # mutates df in-place
df.sem_map("...")        # returns new copy
# df vẫn có _lotus_partition_id, nhưng new_df không có
```

### Risk 4: Column Name Collision
```python
df.sem_map("...", suffix="_map")
df.sem_map("...", suffix="_map")  # Overwrite _map column!
```

## 6. Recommended Pipeline Best Practices

1. **Index sớm**: Gọi `sem_index()` đầu pipeline, trước bất kỳ operator nào cần index
2. **Chain carefully**: Lưu ý attrs không được truyền qua tất cả operators
3. **Custom suffixes**: Dùng unique suffix khi chain nhiều sem_map calls
4. **Filter trước Join**: Giảm N trước khi chạy O(M*N) join
5. **Partition trước Agg**: Dùng sem_partition_by để cải thiện chất lượng aggregation
