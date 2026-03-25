# SEM_SIM_JOIN — `sem_sim_join`

## Metadata
- **File**: `lotus/sem_ops/sem_sim_join.py`
- **Accessor line**: 12 (`@pd.api.extensions.register_dataframe_accessor("sem_sim_join")`)
- **Type**: Binary operator
- **Requires**: `lotus.settings.rm` (RM) + `lotus.settings.vs` (VS) (sem_sim_join.py:101-106)
- **LLM**: Khong dung LLM — pure embedding-based join

## 1. Purpose & Use Cases

Join 2 DataFrames bang embedding similarity. Moi row ben trai duoc match voi K rows ben phai gan nhat trong khong gian embedding.

**Use cases:**
- Category matching: `df1.sem_sim_join(df2, "article", "category", K=1)`
- Similar item finding: `df1.sem_sim_join(df2, "product", "item", K=3)`
- Approximate entity resolution: KNN-based matching

## 2. Call Stack Trace

```
1. SemSimJoinDataframe.__call__()                       # sem_sim_join.py:85
2.   [Neu other la Series]: convert thanh DataFrame      # sem_sim_join.py:96-99
3.   rm = lotus.settings.rm, vs = lotus.settings.vs      # sem_sim_join.py:101-102
4.   [Neu left_on co index]:
4a.    Load left index, get vectors                      # sem_sim_join.py:109-117
4b.    [NotImplementedError]: fallback sang raw values   # sem_sim_join.py:116-117
5.   [Else]: queries = df[left_on]                       # sem_sim_join.py:119
6.   Load right index                                    # sem_sim_join.py:122-128
7.   rm.convert_query_to_query_vector(queries)           # sem_sim_join.py:130
8.   vs(query_vectors, K, ids=right_ids)                 # sem_sim_join.py:134
9.   Post-filter: chi giu valid indices                  # sem_sim_join.py:142-145
10.  Build joined DataFrame                              # sem_sim_join.py:147-164
11.  Return joined DataFrame voi _scores column          # sem_sim_join.py:166
```

## 3. Prompt Template (COPY VERBATIM)

Khong co prompt — sem_sim_join khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** Chi dung:
- **RM**: `rm.convert_query_to_query_vector(queries)` (sem_sim_join.py:130)
- **VS**: `vs(query_vectors, K, ids=right_ids)` (sem_sim_join.py:134) — KNN search trong right index

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Query vectors duoc gui batch qua RM (sem_sim_join.py:130) |
| Caching | Yes | `@operator_cache` (sem_sim_join.py:84) |
| Index reuse | Yes | Load vectors tu existing index (sem_sim_join.py:109-117) |
| Post-filtering | Yes | Filter invalid indices (sem_sim_join.py:142-145) |

## 6. Input/Output Contract

### Input:
- `other: pd.DataFrame` — DataFrame ben phai (sem_sim_join.py:87)
- `left_on: str` — Column ben trai de match (sem_sim_join.py:88)
- `right_on: str` — Column ben phai de match (phai co index) (sem_sim_join.py:89)
- `K: int` — So nearest neighbors cho moi left row (sem_sim_join.py:90)
- `lsuffix/rsuffix: str` — Suffix cho columns trung ten (sem_sim_join.py:91-92)
- `keep_index: bool` — Giu `_left_id`/`_right_id` columns (sem_sim_join.py:94, default=False)

### Output:
- Joined DataFrame: moi left row co K right rows (sem_sim_join.py:166)
- Column `_scores{score_suffix}`: similarity scores (sem_sim_join.py:151)
- Neu `keep_index=True`: giu `_left_id` va `_right_id` columns

### Yeu cau:
- `right_on` column phai duoc index (sem_sim_join.py:122-125)
- `left_on` column co the co hoac khong co index — neu co se dung pre-computed vectors (sem_sim_join.py:109)

## 7. Edge Cases

1. **Right column chua index**: Raise `ValueError` (sem_sim_join.py:124-125)
2. **Left column co index nhung vector store khong ho tro get_vectors**: Fallback sang raw values (sem_sim_join.py:116-117)
3. **K > len(other)**: VS se tra ve tat ca available items
4. **Invalid indices (-1)**: Bi filter out (sem_sim_join.py:144)
5. **Index not in other**: Bi filter out (sem_sim_join.py:144)

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

### Diem manh:
- **Fast**: Pure embedding comparison — khong can LLM calls
- **Scalable**: KNN search efficient cho datasets lon
- **Index reuse**: Co the tai su dung pre-computed vectors tu left index
- **Foundation for cascade**: Duoc sem_join dung lam helper model (sem_join.py:362)

### Diem yeu:
- **Right index required**: Phai index right column truoc — them buoc setup
- **No score normalization standard**: Scores phu thuoc vao RM va VS implementation
- **No threshold filtering**: Khong co built-in threshold — caller phai tu filter theo _scores
- **Column naming**: `_scores` + suffix co the conflict voi existing columns
