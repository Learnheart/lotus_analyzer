# SEM_CLUSTER_BY — `sem_cluster_by`

## Metadata
- **File**: `lotus/sem_ops/sem_cluster_by.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_cluster_by")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_cluster_by.py:67-72)
- **LLM**: Khong dung LLM

## 1. Purpose & Use Cases

Phan cum DataFrame dung FAISS kmeans tren vector embeddings. Column phai duoc index truoc bang sem_index.

**Use cases:**
- Topic clustering: `df.sem_cluster_by("title", ncentroids=5)`
- Data grouping: `df.sem_cluster_by("description", ncentroids=3)`

## 2. Call Stack Trace

```
1. SemClusterByDataframe.__call__()                     # sem_cluster_by.py:58
2.   rm = lotus.settings.rm, vs = lotus.settings.vs     # sem_cluster_by.py:67-68
3.   Validate rm va vs                                   # sem_cluster_by.py:69-72
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

Khong co prompt — sem_cluster_by khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** Chi dung:
- **VS**: `vs.get_vectors_from_index()` (utils.py:59) — lay pre-computed vectors
- **FAISS**: `faiss.Kmeans` — chay kmeans clustering

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Caching | Yes | `@operator_cache` (sem_cluster_by.py:57) |
| Index reuse | Yes | Dung vectors tu existing index (utils.py:59) |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column de cluster (phai co index) (sem_cluster_by.py:60)
- `ncentroids: int` — So clusters (sem_cluster_by.py:61)
- `return_scores: bool` — Commented out (sem_cluster_by.py:62)
- `return_centroids: bool` — Commented out (sem_cluster_by.py:63)
- `niter: int` — So iterations kmeans (sem_cluster_by.py:64, default=20)
- `verbose: bool` — Print kmeans output (sem_cluster_by.py:65, default=False)

### Output:
- DataFrame goc + column `cluster_id` (sem_cluster_by.py:78)

### Yeu cau:
- Column phai duoc index truoc (utils.py:50-52)
- `ncentroids <= len(df)` (utils.py:38-39)

## 7. Edge Cases

1. **Column chua index**: Raise `ValueError` (utils.py:51-52)
2. **ncentroids > len(df)**: Raise `ValueError` (utils.py:38-39)
3. **RM/VS chua configure**: Raise `ValueError` (sem_cluster_by.py:69-72)
4. **return_scores/return_centroids**: Commented out (sem_cluster_by.py:79-85) — chua implement

## 8. Code Examples

```python
# Setup
df = df.sem_index("title", "title_index")

# Basic clustering
df = df.sem_cluster_by("title", ncentroids=3)

# Voi custom iterations
df = df.sem_cluster_by("title", ncentroids=5, niter=50, verbose=True)
```

## 9. Assessment

### Diem manh:
- **Efficient**: FAISS kmeans rat nhanh cho clustering
- **Simple API**: Chi can col_name va ncentroids
- **Index reuse**: Dung pre-computed vectors

### Diem yeu:
- **Commented-out features**: return_scores va return_centroids chua implement (sem_cluster_by.py:79-85)
- **Fixed algorithm**: Chi ho tro kmeans — khong co DBSCAN, hierarchical, etc.
- **Mutates in-place**: Them `cluster_id` truc tiep vao self._obj (sem_cluster_by.py:78)
- **Coupling voi utils.cluster**: Logic clustering o utils.py, khong o operator file
- **Hardcoded column name**: `cluster_id` la hardcoded — khong co option custom
