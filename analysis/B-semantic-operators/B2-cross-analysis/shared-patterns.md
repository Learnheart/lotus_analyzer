# B2 — Shared Patterns Across Operators

> **Phân tích** các patterns chung giữa tất cả semantic operators trong LOTUS.

## 1. DataFrame Accessor Pattern

**Tất cả** operators dùng `@pd.api.extensions.register_dataframe_accessor`:

| Operator | Registration | File:Line |
|---|---|---|
| sem_filter | `@pd.api.extensions.register_dataframe_accessor("sem_filter")` | sem_filter.py:225 |
| sem_map | `@pd.api.extensions.register_dataframe_accessor("sem_map")` | sem_map.py:121 |
| sem_join | `@pd.api.extensions.register_dataframe_accessor("sem_join")` | sem_join.py:606 |
| sem_agg | `@pd.api.extensions.register_dataframe_accessor("sem_agg")` | sem_agg.py:226 |
| sem_topk | `@pd.api.extensions.register_dataframe_accessor("sem_topk")` | sem_topk.py:624 |
| sem_extract | `@pd.api.extensions.register_dataframe_accessor("sem_extract")` | sem_extract.py:111 |
| sem_search | `@pd.api.extensions.register_dataframe_accessor("sem_search")` | sem_search.py:10 |
| sem_sim_join | `@pd.api.extensions.register_dataframe_accessor("sem_sim_join")` | sem_sim_join.py:12 |
| sem_dedup | `@pd.api.extensions.register_dataframe_accessor("sem_dedup")` | sem_dedup.py:10 |
| sem_index | `@pd.api.extensions.register_dataframe_accessor("sem_index")` | sem_index.py:9 |
| sem_partition_by | `@pd.api.extensions.register_dataframe_accessor("sem_partition_by")` | sem_partition_by.py:8 |
| sem_cluster_by | `@pd.api.extensions.register_dataframe_accessor("sem_cluster_by")` | sem_cluster_by.py:10 |

**Pattern structure**: Mỗi accessor class có:
- `__init__(self, pandas_obj)` — lưu `self._obj = pandas_obj`
- `_validate(obj)` — static method kiểm tra input là DataFrame
- `__call__(self, ...)` — logic chính, được gọi khi user dùng `df.sem_xxx(...)`

## 2. `@operator_cache` Decorator

**Tất cả** operators dùng `@operator_cache` (cache.py:33-100) trên `__call__`:

| Operator | Cache line |
|---|---|
| sem_filter | sem_filter.py:333 |
| sem_map | sem_map.py:214 |
| sem_join | sem_join.py:669 |
| sem_agg | sem_agg.py:353 |
| sem_topk | sem_topk.py:734 |
| sem_extract | sem_extract.py:201 |
| sem_search | sem_search.py:91 |
| sem_sim_join | sem_sim_join.py:84 |
| sem_dedup | sem_dedup.py:32 |
| sem_index | sem_index.py:61 |
| sem_partition_by | sem_partition_by.py:60 |
| sem_cluster_by | sem_cluster_by.py:57 |

**Cách hoạt động** (cache.py:33-100):
1. Serialize `self._obj` (DataFrame), args, kwargs thành JSON (cache.py:69-71)
2. Hash SHA-256 làm cache key (cache.py:72-76)
3. Check cache hit (cache.py:79-88)
4. Nếu miss: chạy function, lưu result + virtual_usage vào cache (cache.py:91-96)

## 3. Langex Parsing (`parse_cols`)

**Hầu hết** LLM-based operators dùng `lotus.nl_expression.parse_cols()` (nl_expression.py:4):

| Operator | parse_cols usage | File:Line |
|---|---|---|
| sem_filter | `col_li = lotus.nl_expression.parse_cols(user_instruction)` | sem_filter.py:358 |
| sem_map | `col_li = lotus.nl_expression.parse_cols(user_instruction)` | sem_map.py:234 |
| sem_join | `cols = lotus.nl_expression.parse_cols(join_instruction)` | sem_join.py:699 |
| sem_agg | `col_li = lotus.nl_expression.parse_cols(user_instruction)` | sem_agg.py:373 |
| sem_topk | `col_li = lotus.nl_expression.parse_cols(user_instruction)` | sem_topk.py:754 |
| sem_extract | Không dùng — dùng `input_cols` parameter trực tiếp | — |

**Ngoại lệ**: sem_extract dùng `input_cols` list thay vì langex (sem_extract.py:203). Tất cả embedding-based operators (sem_search, sem_sim_join, sem_dedup, sem_index, sem_cluster_by, sem_partition_by) không dùng parse_cols vì không có langex.

## 4. Data Serialization (`df2multimodal_info`)

**Hầu hết** LLM-based operators dùng `task_instructions.df2multimodal_info()` (task_instructions.py:364-379):

| Operator | df2multimodal_info usage | File:Line |
|---|---|---|
| sem_filter | `multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)` | sem_filter.py:367 |
| sem_map | `multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)` | sem_map.py:241 |
| sem_join | `task_instructions.df2multimodal_info(l1.to_frame(...), ...)` | sem_join.py:101-102 |
| sem_topk | `multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)` | sem_topk.py:790 |
| sem_extract | `multimodal_data = task_instructions.df2multimodal_info(self._obj, input_cols)` | sem_extract.py:226 |

**Ngoại lệ**: sem_agg dùng `df2text()` hoặc `create_chunked_documents()` tùy theo strategy (sem_agg.py:418-427).

**df2multimodal_info flow** (task_instructions.py:364-379):
1. Tách image columns và text columns (task_instructions.py:369-370)
2. Format text columns qua `df2text()` (task_instructions.py:371)
3. Get base64 images cho image columns (task_instructions.py:375)
4. Return `[{"text": ..., "image": {...}}]` per row

## 5. LM Check Pattern

**Tất cả** LLM-based operators kiểm tra `lotus.settings.lm is not None`:

| Operator | Check | File:Line |
|---|---|---|
| sem_filter | `if lotus.settings.lm is None: raise ValueError(...)` | sem_filter.py:351-354 |
| sem_map | `if lotus.settings.lm is None: raise ValueError(...)` | sem_map.py:229-232 |
| sem_join | `if model is None: raise ValueError(...)` | sem_join.py:686-689 |
| sem_agg | `if lotus.settings.lm is None: raise ValueError(...)` | sem_agg.py:364-367 |
| sem_topk | `if model is None: raise ValueError(...)` | sem_topk.py:748-751 |
| sem_extract | `if lotus.settings.lm is None: raise ValueError(...)` | sem_extract.py:216-219 |

**Error message chung**: `"The language model must be an instance of LM. Please configure a valid language model using lotus.settings.configure()"`

## 6. RM/VS Check Pattern

**Tất cả** embedding-based operators kiểm tra RM và VS:

| Operator | Check | File:Line |
|---|---|---|
| sem_search | `if rm is None or vs is None: raise ValueError(...)` | sem_search.py:106-109 |
| sem_sim_join | `if not isinstance(rm, RM) or not isinstance(vs, VS): raise ValueError(...)` | sem_sim_join.py:103-106 |
| sem_dedup | `if rm is None or vs is None: raise ValueError(...)` | sem_dedup.py:40-43 |
| sem_index | `if rm is None or vs is None: raise ValueError(...)` | sem_index.py:69-72 |
| sem_cluster_by | `if rm is None or vs is None: raise ValueError(...)` | sem_cluster_by.py:69-72 |

**Lưu ý**: sem_sim_join dùng `isinstance` check (sem_sim_join.py:103), các operator khác dùng `is None` check — inconsistent.

## 7. Column Validation Pattern

**Pattern chung**: Loop qua columns và kiểm tra tồn tại trong DataFrame:

```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

| Operator | Validation | File:Line |
|---|---|---|
| sem_filter | Loop check | sem_filter.py:363-365 |
| sem_map | Loop check | sem_map.py:237-239 |
| sem_agg | Loop check | sem_agg.py:377-379 |
| sem_topk | Loop check | sem_topk.py:758-760 |
| sem_extract | Loop check | sem_extract.py:222-224 |
| sem_join | Implicit qua column lookup | sem_join.py:700-730 |

## 8. `nle2str` Formatting Pattern

**LLM operators** dùng `lotus.nl_expression.nle2str()` (nl_expression.py:17-21) để format instruction:

| Operator | nle2str usage | File:Line |
|---|---|---|
| sem_filter | `formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)` | sem_filter.py:369 |
| sem_map | `formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)` | sem_map.py:242 |
| sem_agg | `formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)` | sem_agg.py:408 |
| sem_topk | `formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)` | sem_topk.py:792 |

**Lưu ý**: sem_join không dùng nle2str — truyền join_instruction trực tiếp vì cần giữ {col} format cho merge_multimodal_info.

## 9. Mutability Pattern

**Các operators có mutate self._obj**:
- `sem_index`: `self._obj.attrs["index_dirs"][col_name] = index_dir` (sem_index.py:76)
- `sem_partition_by`: `self._obj["_lotus_partition_id"] = ...` (sem_partition_by.py:66)
- `sem_cluster_by`: `self._obj["cluster_id"] = ...` (sem_cluster_by.py:78)
- `sem_agg`: `self._obj = self._obj.sort_values(...)` (sem_agg.py:403)

**Các operators trả về new DataFrame** (không mutate):
- `sem_filter`: `new_df = self._obj.iloc[ids]` hoặc `new_df = self._obj.copy()` (sem_filter.py:563, 576)
- `sem_map`: `new_df = self._obj.copy()` (sem_map.py:272)
- `sem_extract`: `new_df = self._obj.copy()` (sem_extract.py:240)

**Inconsistency**: Một số operators mutate, một số copy — behavior không nhất quán.
