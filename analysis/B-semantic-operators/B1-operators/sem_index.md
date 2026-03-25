# SEM_INDEX — `sem_index`

## Metadata
- **File**: `lotus/sem_ops/sem_index.py`
- **Accessor line**: 9 (`@pd.api.extensions.register_dataframe_accessor("sem_index")`)
- **Type**: Unary (side-effect)
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_index.py:67-72)
- **LLM**: Khong dung LLM

## 1. Purpose & Use Cases

Tao vector similarity index cho 1 column trong DataFrame. Day la prerequisite cho sem_search, sem_sim_join, sem_dedup, sem_cluster_by, va cascade modes cua sem_filter/sem_join.

**Use cases:**
- Index for search: `df.sem_index("title", "title_index")`
- Index for join: `df.sem_index("category", "cat_index")`
- Index for cluster: `df.sem_index("text", "text_index")`

## 2. Call Stack Trace

```
1. SemIndexDataframe.__call__()                         # sem_index.py:62
2.   Warning: khong reset DataFrame index               # sem_index.py:63-65
3.   rm = lotus.settings.rm, vs = lotus.settings.vs     # sem_index.py:67-68
4.   Validate rm va vs                                   # sem_index.py:69-72
5.   embeddings = rm(df[col_name].tolist())              # sem_index.py:74
6.   vs.index(df[col_name], embeddings, index_dir)       # sem_index.py:75
7.   df.attrs["index_dirs"][col_name] = index_dir        # sem_index.py:76
8.   Return df (unchanged, but index created)            # sem_index.py:77
```

## 3. Prompt Template (COPY VERBATIM)

Khong co prompt — sem_index khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** Chi dung:
- **RM**: `rm(df[col_name].tolist())` (sem_index.py:74) — compute embeddings cho tat ca values
- **VS**: `vs.index(df[col_name], embeddings, index_dir)` (sem_index.py:75) — xay index va luu vao `index_dir`

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | RM tinh embeddings batch (sem_index.py:74) |
| Caching | Yes | `@operator_cache` (sem_index.py:61) |
| Persistence | Yes | Index duoc luu vao disk tai index_dir (sem_index.py:75) |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column de index (sem_index.py:62)
- `index_dir: str` — Thu muc luu index (sem_index.py:62)

### Output:
- DataFrame giong het input (sem_index.py:77)
- **Side effect**: `df.attrs["index_dirs"][col_name] = index_dir` (sem_index.py:76)
- **Side effect**: Index file duoc tao tai `index_dir` tren disk

### Init side effect:
Constructor `__init__` tao `self._obj.attrs["index_dirs"] = {}` (sem_index.py:54) — **chu y**: co the xoa index_dirs cua DataFrame khac!

## 7. Edge Cases

1. **RM/VS chua configure**: Raise `ValueError` (sem_index.py:69-72)
2. **DataFrame index da bi reset**: Warning message (sem_index.py:63-65) — `get_vectors_from_index` co the fail
3. **Column khong ton tai**: KeyError tu `df[col_name]` (sem_index.py:74)
4. **Index_dir da ton tai**: Behavior phu thuoc vao VS implementation — co the overwrite
5. **Init overwrite**: `__init__` set `attrs["index_dirs"] = {}` (sem_index.py:54) — moi lan access `.sem_index` se reset dict!

## 8. Code Examples

```python
# Create index
df = df.sem_index("title", "title_index")

# Sau do co the dung:
df.sem_search("title", "AI", K=5)
df.sem_sim_join(other_df, "title", "category", K=1)
df.sem_cluster_by("title", ncentroids=3)

# Load index tu disk
df.load_sem_index("title", "title_index")
```

## 9. Assessment

### Diem manh:
- **Persistence**: Index luu disk, chi can tao 1 lan
- **Simple API**: Chi can col_name va index_dir
- **Foundation operator**: Can thiet cho nhieu operators khac

### Diem yeu:
- **Init bug potential**: `__init__` set `attrs["index_dirs"] = {}` (sem_index.py:54) — neu DataFrame da co index_dirs, se bi mat khi access `.sem_index`
- **No incremental update**: Phai rebuild toan bo index khi data thay doi
- **No validation**: Khong kiem tra col_name co ton tai khong truoc khi compute embeddings
- **Warning only**: "Do not reset the dataframe index" chi la warning (sem_index.py:63-65) — khong enforce
