# SEM_DEDUP — `sem_dedup`

## Metadata
- **File**: `lotus/sem_ops/sem_dedup.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_dedup")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_dedup.py:38-43)
- **LLM**: Không dùng LLM — pure embedding-based

## 1. Purpose & Use Cases

Loại bỏ các duplicate rows dựa trên embedding similarity. Dùng connected components algorithm để xử lý transitive duplicates (A~B, B~C => loại C).

**Use cases:**
- Data deduplication: `df.sem_dedup("title", threshold=0.9)`
- Near-duplicate removal: `df.sem_dedup("content", threshold=0.85)`

## 2. Call Stack Trace

```
1. SemDedupByDataframe.__call__()                       # sem_dedup.py:33
2.   Validate rm và vs                                   # sem_dedup.py:38-43
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

Không có prompt — sem_dedup không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** Chỉ dùng sem_sim_join nội bộ (sem_dedup.py:45).

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | N/A | Embedding-based |
| Caching | Yes | `@operator_cache` (sem_dedup.py:32) |
| Cascading | No | |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column để dedup (sem_dedup.py:36)
- `threshold: float` — Similarity threshold (sem_dedup.py:37)

### Output:
- DataFrame với duplicate rows đã bị loại (sem_dedup.py:91)
- Giữ lại row đầu tiên của mỗi connected component (sem_dedup.py:88-89)

### Yêu cầu:
- Column phải được index trước bằng sem_index()

## 7. Edge Cases

1. **Threshold quá thấp**: Nhiều rows bị group lại, mất nhiều data
2. **Threshold quá cao**: Không loại được duplicate nào
3. **Self-join**: sem_sim_join với chính nó, K=len(df) (sem_dedup.py:45) — O(N^2) comparisons
4. **Transitive dedup**: A~B, B~C => A, B, C trong 1 component, chỉ giữ A (sem_dedup.py:88-89)
5. **Value-based filtering**: Filter theo `col_name` values, không theo index (sem_dedup.py:47, 91) — có thể có vấn đề nếu nhiều rows có cùng value

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

### Điểm mạnh:
- **Transitive dedup**: Connected components xử lý transitive similarity đúng cách
- **No LLM cost**: Pure embedding-based — nhanh và rẻ
- **Simple API**: Chỉ cần col_name và threshold

### Điểm yếu:
- **O(N^2) memory**: Self-join với K=len(df) (sem_dedup.py:45) — không scale cho datasets lớn
- **Value-based comparison**: Filter dùng value equality (`col_l != col_r`) tại sem_dedup.py:47 — không phải index-based, có thể sai khi nhiều rows có cùng value
- **First-wins strategy**: Giữ row đầu tiên của component (sem_dedup.py:88-89) — không có option chọn representative
- **No cluster info output**: Không trả về cluster/component information
- **DFS stack-based**: `find_connected_components` dùng iterative DFS (sem_dedup.py:67-74) — tốt cho avoiding recursion limit
