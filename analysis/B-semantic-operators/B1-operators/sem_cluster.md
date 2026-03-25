# SEM_CLUSTER_BY — `sem_cluster_by`

## Metadata
- **File**: `lotus/sem_ops/sem_cluster_by.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_cluster_by")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_cluster_by.py:67-72)
- **LLM**: Không dùng LLM

## 1. Purpose & Use Cases

Phân cụm DataFrame dùng FAISS kmeans trên vector embeddings. Column phải được index trước bằng sem_index.

**Use cases:**
- Topic clustering: `df.sem_cluster_by("title", ncentroids=5)`
- Data grouping: `df.sem_cluster_by("description", ncentroids=3)`

## 2. Call Stack Trace

```
1. SemClusterByDataframe.__call__()                     # sem_cluster_by.py:58
2.   rm = lotus.settings.rm, vs = lotus.settings.vs     # sem_cluster_by.py:67-68
3.   Validate rm và vs                                   # sem_cluster_by.py:69-72
4.   cluster_fn = lotus.utils.cluster(col_name, n)       # sem_cluster_by.py:74
5.   indices = cluster_fn(self._obj, niter, verbose)     # sem_cluster_by.py:76
6.   [Trong cluster_fn - utils.py:14-60]:
6a.    Load index: vs.load_index(col_index_dir)          # utils.py:54-56
6b.    Get vectors: vs.get_vectors_from_index()           # utils.py:59
6c.    faiss.Kmeans(d, ncentroids, niter, verbose)        # utils.py (faiss)
6d.    kmeans.train(vec_set)                              # utils.py (faiss)
6e.    Return cluster assignments
7.   df["cluster_id"] = Series(indices)                  # sem_cluster_by.py:78
8.   Return df                                          # sem_cluster_by.py:86
```

## 3. Prompt Template (COPY VERBATIM)

Không có prompt — sem_cluster_by không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** Chỉ dùng:
- **VS**: `vs.get_vectors_from_index()` (utils.py:59) — lấy pre-computed vectors
- **FAISS**: `faiss.Kmeans` — chạy kmeans clustering

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Caching | Yes | `@operator_cache` (sem_cluster_by.py:57) |
| Index reuse | Yes | Dùng vectors từ existing index (utils.py:59) |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column để cluster (phải có index) (sem_cluster_by.py:60)
- `ncentroids: int` — Số clusters (sem_cluster_by.py:61)
- `return_scores: bool` — Commented out (sem_cluster_by.py:62)
- `return_centroids: bool` — Commented out (sem_cluster_by.py:63)
- `niter: int` — Số iterations kmeans (sem_cluster_by.py:64, default=20)
- `verbose: bool` — Print kmeans output (sem_cluster_by.py:65, default=False)

### Output:
- DataFrame gốc + column `cluster_id` (sem_cluster_by.py:78)

### Yêu cầu:
- Column phải được index trước (utils.py:50-52)
- `ncentroids <= len(df)` (utils.py:38-39)

## 7. Edge Cases

1. **Column chưa index**: Raise `ValueError` (utils.py:51-52)
2. **ncentroids > len(df)**: Raise `ValueError` (utils.py:38-39)
3. **RM/VS chưa configure**: Raise `ValueError` (sem_cluster_by.py:69-72)
4. **return_scores/return_centroids**: Commented out (sem_cluster_by.py:79-85) — chưa implement

## 8. Code Examples

```python
# Setup
df = df.sem_index("title", "title_index")

# Basic clustering
df = df.sem_cluster_by("title", ncentroids=3)

# Với custom iterations
df = df.sem_cluster_by("title", ncentroids=5, niter=50, verbose=True)
```

## 9. Assessment

### Điểm mạnh:
- **Efficient**: FAISS kmeans rất nhanh cho clustering
- **Simple API**: Chỉ cần col_name và ncentroids
- **Index reuse**: Dùng pre-computed vectors

### Điểm yếu:
- **Commented-out features**: return_scores và return_centroids chưa implement (sem_cluster_by.py:79-85)
- **Fixed algorithm**: Chỉ hỗ trợ kmeans — không có DBSCAN, hierarchical, etc.
- **Mutates in-place**: Thêm `cluster_id` trực tiếp vào self._obj (sem_cluster_by.py:78)
- **Coupling với utils.cluster**: Logic clustering ở utils.py, không ở operator file
- **Hardcoded column name**: `cluster_id` là hardcoded — không có option custom
