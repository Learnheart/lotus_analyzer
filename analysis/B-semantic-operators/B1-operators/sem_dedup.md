# SEM_DEDUP — `sem_dedup`

## Metadata
- **File**: `lotus/sem_ops/sem_dedup.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_dedup")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_dedup.py:38-43)
- **LLM**: Khong dung LLM — pure embedding-based

## 1. Purpose & Use Cases

Loai bo cac duplicate rows dua tren embedding similarity. Dung connected components algorithm de xu ly transitive duplicates (A~B, B~C => loai C).

**Use cases:**
- Data deduplication: `df.sem_dedup("title", threshold=0.9)`
- Near-duplicate removal: `df.sem_dedup("content", threshold=0.85)`

## 2. Call Stack Trace

```
1. SemDedupByDataframe.__call__()                       # sem_dedup.py:33
2.   Validate rm va vs                                   # sem_dedup.py:38-43
3.   self._obj.sem_sim_join(self._obj, col, col, len)   # sem_dedup.py:45
4.   Filter theo threshold: _scores > threshold          # sem_dedup.py:46
5.   Remove self-matches: left != right                  # sem_dedup.py:47
6.   Build pairs set                                     # sem_dedup.py:51-56
7.   find_connected_components(pairs)                    # sem_dedup.py:84
7a.    Build adjacency graph                             # sem_dedup.py:60-62
7b.    DFS traversal                                     # sem_dedup.py:67-74
7c.    Return components                                 # sem_dedup.py:82
8.   Remove all but first element of each component      # sem_dedup.py:88-89
9.   Return filtered DataFrame                           # sem_dedup.py:91
```

## 3. Prompt Template (COPY VERBATIM)

Khong co prompt — sem_dedup khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** Chi dung sem_sim_join noi bo (sem_dedup.py:45).

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | N/A | Embedding-based |
| Caching | Yes | `@operator_cache` (sem_dedup.py:32) |
| Cascading | No | |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column de dedup (sem_dedup.py:36)
- `threshold: float` — Similarity threshold (sem_dedup.py:37)

### Output:
- DataFrame voi duplicate rows da bi loai (sem_dedup.py:91)
- Giu lai row dau tien cua moi connected component (sem_dedup.py:88-89)

### Yeu cau:
- Column phai duoc index truoc bang sem_index()

## 7. Edge Cases

1. **Threshold qua thap**: Nhieu rows bi group lai, mat nhieu data
2. **Threshold qua cao**: Khong loai duoc duplicate nao
3. **Self-join**: sem_sim_join voi chinh no, K=len(df) (sem_dedup.py:45) — O(N^2) comparisons
4. **Transitive dedup**: A~B, B~C => A, B, C trong 1 component, chi giu A (sem_dedup.py:88-89)
5. **Value-based filtering**: Filter theo `col_name` values, khong theo index (sem_dedup.py:47, 91) — co the co van de neu nhieu rows co cung value

## 8. Code Examples

```python
# Setup
df = df.sem_index("title", "title_index")

# Basic dedup
df.sem_dedup("title", threshold=0.9)

# Stricter threshold
df.sem_dedup("title", threshold=0.95)
```

## 9. Assessment

### Diem manh:
- **Transitive dedup**: Connected components xu ly transitive similarity dung cach
- **No LLM cost**: Pure embedding-based — nhanh va re
- **Simple API**: Chi can col_name va threshold

### Diem yeu:
- **O(N^2) memory**: Self-join voi K=len(df) (sem_dedup.py:45) — khong scale cho datasets lon
- **Value-based comparison**: Filter dung value equality (`col_l != col_r`) tai sem_dedup.py:47 — khong phai index-based, co the sai khi nhieu rows co cung value
- **First-wins strategy**: Giu row dau tien cua component (sem_dedup.py:88-89) — khong co option chon representative
- **No cluster info output**: Khong tra ve cluster/component information
- **DFS stack-based**: `find_connected_components` dung iterative DFS (sem_dedup.py:67-74) — tot cho avoiding recursion limit
