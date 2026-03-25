# SEM_SIM_JOIN — `sem_sim_join`

## Metadata
- **File**: `lotus/sem_ops/sem_sim_join.py`
- **Accessor line**: 12 (`@pd.api.extensions.register_dataframe_accessor("sem_sim_join")`)
- **Type**: Binary operator
- **Requires**: `lotus.settings.rm` (RM) + `lotus.settings.vs` (VS) (sem_sim_join.py:101-106)
- **LLM**: Không dùng LLM — pure embedding-based join

## 1. Purpose & Use Cases

Join 2 DataFrames bằng embedding similarity. Mỗi row bên trái được match với K rows bên phải gần nhất trong không gian embedding.

**Use cases:**
- Category matching: `df1.sem_sim_join(df2, "article", "category", K=1)`
- Similar item finding: `df1.sem_sim_join(df2, "product", "item", K=3)`
- Approximate entity resolution: KNN-based matching

## 2. Call Stack Trace

```
1. SemSimJoinDataframe.__call__()                       # sem_sim_join.py:85
2.   [Nếu other là Series]: convert thành DataFrame      # sem_sim_join.py:96-99
3.   rm = lotus.settings.rm, vs = lotus.settings.vs      # sem_sim_join.py:101-102
4.   [Nếu left_on có index]:
4a.    Load left index, get vectors                      # sem_sim_join.py:109-117
4b.    [NotImplementedError]: fallback sang raw values   # sem_sim_join.py:116-117
5.   [Else]: queries = df[left_on]                       # sem_sim_join.py:119
6.   Load right index                                    # sem_sim_join.py:122-128
7.   rm.convert_query_to_query_vector(queries)           # sem_sim_join.py:130
8.   vs(query_vectors, K, ids=right_ids)                 # sem_sim_join.py:134
9.   Post-filter: chỉ giữ valid indices                  # sem_sim_join.py:142-145
10.  Build joined DataFrame                              # sem_sim_join.py:147-164
11.  Return joined DataFrame với _scores column          # sem_sim_join.py:166
```

## 3. Prompt Template (COPY VERBATIM)

Không có prompt — sem_sim_join không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** Chỉ dùng:
- **RM**: `rm.convert_query_to_query_vector(queries)` (sem_sim_join.py:130)
- **VS**: `vs(query_vectors, K, ids=right_ids)` (sem_sim_join.py:134) — KNN search trong right index

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | Query vectors được gửi batch qua RM (sem_sim_join.py:130) |
| Caching | Yes | `@operator_cache` (sem_sim_join.py:84) |
| Index reuse | Yes | Load vectors từ existing index (sem_sim_join.py:109-117) |
| Post-filtering | Yes | Filter invalid indices (sem_sim_join.py:142-145) |

## 6. Input/Output Contract

### Input:
- `other: pd.DataFrame` — DataFrame bên phải (sem_sim_join.py:87)
- `left_on: str` — Column bên trái để match (sem_sim_join.py:88)
- `right_on: str` — Column bên phải để match (phải có index) (sem_sim_join.py:89)
- `K: int` — Số nearest neighbors cho mỗi left row (sem_sim_join.py:90)
- `lsuffix/rsuffix: str` — Suffix cho columns trùng tên (sem_sim_join.py:91-92)
- `keep_index: bool` — Giữ `_left_id`/`_right_id` columns (sem_sim_join.py:94, default=False)

### Output:
- Joined DataFrame: mỗi left row có K right rows (sem_sim_join.py:166)
- Column `_scores{score_suffix}`: similarity scores (sem_sim_join.py:151)
- Nếu `keep_index=True`: giữ `_left_id` và `_right_id` columns

### Yêu cầu:
- `right_on` column phải được index (sem_sim_join.py:122-125)
- `left_on` column có thể có hoặc không có index — nếu có sẽ dùng pre-computed vectors (sem_sim_join.py:109)

## 7. Edge Cases

1. **Right column chưa index**: Raise `ValueError` (sem_sim_join.py:124-125)
2. **Left column có index nhưng vector store không hỗ trợ get_vectors**: Fallback sang raw values (sem_sim_join.py:116-117)
3. **K > len(other)**: VS sẽ trả về tất cả available items
4. **Invalid indices (-1)**: Bị filter out (sem_sim_join.py:144)
5. **Index not in other**: Bị filter out (sem_sim_join.py:144)

## 8. Code Examples

```python
# Setup
df1 = pd.DataFrame({"article": ["ML tutorial", "Cooking guide"]})
df2 = pd.DataFrame({"category": ["CS", "AI", "Cooking"]})
df2 = df2.sem_index("category", "cat_index")

# Basic sim join - K=1
df1.sem_sim_join(df2, "article", "category", K=1)

# K=2 - top 2 matches per article
df1.sem_sim_join(df2, "article", "category", K=2)

# Keep index columns
df1.sem_sim_join(df2, "article", "category", K=1, keep_index=True)
```

## 9. Assessment

### Điểm mạnh:
- **Fast**: Pure embedding comparison — không cần LLM calls
- **Scalable**: KNN search efficient cho datasets lớn
- **Index reuse**: Có thể tái sử dụng pre-computed vectors từ left index
- **Foundation for cascade**: Được sem_join dùng làm helper model (sem_join.py:362)

### Điểm yếu:
- **Right index required**: Phải index right column trước — thêm bước setup
- **No score normalization standard**: Scores phụ thuộc vào RM và VS implementation
- **No threshold filtering**: Không có built-in threshold — caller phải tự filter theo _scores
- **Column naming**: `_scores` + suffix có thể conflict với existing columns
