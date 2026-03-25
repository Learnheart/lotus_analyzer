# SEM_INDEX — `sem_index`

## Metadata
- **File**: `lotus/sem_ops/sem_index.py`
- **Accessor line**: 9 (`@pd.api.extensions.register_dataframe_accessor("sem_index")`)
- **Type**: Unary (side-effect)
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_index.py:67-72)
- **LLM**: Không dùng LLM

## 1. Purpose & Use Cases

Tạo vector similarity index cho 1 column trong DataFrame. Đây là prerequisite cho sem_search, sem_sim_join, sem_dedup, sem_cluster_by, và cascade modes của sem_filter/sem_join.

**Use cases:**
- Index for search: `df.sem_index("title", "title_index")`
- Index for join: `df.sem_index("category", "cat_index")`
- Index for cluster: `df.sem_index("text", "text_index")`

## 2. Call Stack Trace

```
1. SemIndexDataframe.__call__()                         # sem_index.py:62
2.   Warning: không reset DataFrame index               # sem_index.py:63-65
3.   rm = lotus.settings.rm, vs = lotus.settings.vs     # sem_index.py:67-68
4.   Validate rm và vs                                   # sem_index.py:69-72
5.   embeddings = rm(df[col_name].tolist())              # sem_index.py:74
6.   vs.index(df[col_name], embeddings, index_dir)       # sem_index.py:75
7.   df.attrs["index_dirs"][col_name] = index_dir        # sem_index.py:76
8.   Return df (unchanged, but index created)            # sem_index.py:77
```

## 3. Prompt Template (COPY VERBATIM)

Không có prompt — sem_index không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** Chỉ dùng:
- **RM**: `rm(df[col_name].tolist())` (sem_index.py:74) — compute embeddings cho tất cả values
- **VS**: `vs.index(df[col_name], embeddings, index_dir)` (sem_index.py:75) — xây index và lưu vào `index_dir`

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | RM tính embeddings batch (sem_index.py:74) |
| Caching | Yes | `@operator_cache` (sem_index.py:61) |
| Persistence | Yes | Index được lưu vào disk tại index_dir (sem_index.py:75) |

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column để index (sem_index.py:62)
- `index_dir: str` — Thư mục lưu index (sem_index.py:62)

### Output:
- DataFrame giống hệt input (sem_index.py:77)
- **Side effect**: `df.attrs["index_dirs"][col_name] = index_dir` (sem_index.py:76)
- **Side effect**: Index file được tạo tại `index_dir` trên disk

### Init side effect:
Constructor `__init__` tạo `self._obj.attrs["index_dirs"] = {}` (sem_index.py:54) — **chú ý**: có thể xóa index_dirs của DataFrame khác!

## 7. Edge Cases

1. **RM/VS chưa configure**: Raise `ValueError` (sem_index.py:69-72)
2. **DataFrame index đã bị reset**: Warning message (sem_index.py:63-65) — `get_vectors_from_index` có thể fail
3. **Column không tồn tại**: KeyError từ `df[col_name]` (sem_index.py:74)
4. **Index_dir đã tồn tại**: Behavior phụ thuộc vào VS implementation — có thể overwrite
5. **Init overwrite**: `__init__` set `attrs["index_dirs"] = {}` (sem_index.py:54) — mỗi lần access `.sem_index` sẽ reset dict!

## 8. Code Examples

```python
# Create index
df = df.sem_index("title", "title_index")

# Sau đó có thể dùng:
df.sem_search("title", "AI", K=5)
df.sem_sim_join(other_df, "title", "category", K=1)
df.sem_cluster_by("title", ncentroids=3)

# Load index từ disk
df.load_sem_index("title", "title_index")
```

## 9. Assessment

### Điểm mạnh:
- **Persistence**: Index lưu disk, chỉ cần tạo 1 lần
- **Simple API**: Chỉ cần col_name và index_dir
- **Foundation operator**: Cần thiết cho nhiều operators khác

### Điểm yếu:
- **Init bug potential**: `__init__` set `attrs["index_dirs"] = {}` (sem_index.py:54) — nếu DataFrame đã có index_dirs, sẽ bị mất khi access `.sem_index`
- **No incremental update**: Phải rebuild toàn bộ index khi data thay đổi
- **No validation**: Không kiểm tra col_name có tồn tại không trước khi compute embeddings
- **Warning only**: "Do not reset the dataframe index" chỉ là warning (sem_index.py:63-65) — không enforce
