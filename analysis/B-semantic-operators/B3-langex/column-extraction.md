# B3 — Column Extraction & Validation

> **Phân tích** cách columns được trích xuất từ langex và validated trong mỗi operator.

## 1. Extraction Pipeline

### Giai đoạn 1: Parse columns từ langex

```python
# nl_expression.py:4-14
col_li = lotus.nl_expression.parse_cols(user_instruction)
```

Regex `r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"` trích xuất tất cả `{col}` patterns.

**Input**: `"The {text} about {topic} is positive"`
**Output**: `["text", "topic"]`

### Giai đoạn 2: Validate columns tồn tại trong DataFrame

Mỗi operator có validation loop riêng:

**sem_filter** (sem_filter.py:363-365):
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

**sem_map** (sem_map.py:237-239):
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

**sem_agg** (sem_agg.py:377-379):
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"column {column} not found in DataFrame. Given usr instruction: {user_instruction}")
```
Lưu ý: sem_agg có error message khác — thêm instruction vào message.

**sem_topk** (sem_topk.py:758-760):
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"column {column} not found in DataFrame. Given usr instruction: {user_instruction}")
```

**sem_extract** (sem_extract.py:222-224):
```python
for column in input_cols:  # Không dùng parse_cols!
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

### Giai đoạn 3: Data serialization

```python
# task_instructions.py:364-379
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
```

Chỉ serialize các columns được reference — không serialize toàn bộ DataFrame.

## 2. Special Cases: sem_join Column Resolution

sem_join có logic phức tạp hơn để resolve columns giữa 2 DataFrames (sem_join.py:699-730):

```python
cols = lotus.nl_expression.parse_cols(join_instruction)
left_on = None
right_on = None

# Case 1: Explicit :left/:right suffix
for col in cols:
    if ":left" in col:
        left_on = col
        real_left_on = col.split(":left")[0]
    elif ":right" in col:
        right_on = col
        real_right_on = col.split(":right")[0]

# Case 2: Auto-detect based on which DataFrame contains the column
if left_on is None:
    for col in cols:
        if col in self._obj.columns:
            left_on = col
            if col in other.columns:
                raise ValueError("Column found in both dataframes")

if right_on is None:
    for col in cols:
        if col in other.columns:
            right_on = col
            if col in self._obj.columns:
                raise ValueError("Column found in both dataframes")
```

**Resolution rules**:
1. Nếu có `:left`/`:right` suffix → dùng trực tiếp
2. Nếu không → tìm column trong left DF trước, rồi right DF
3. Nếu column tồn tại trong cả 2 DFs → Raise `ValueError`

## 3. Extraction per Operator

| Operator | Column Source | Validation | File:Line |
|---|---|---|---|
| sem_filter | `parse_cols(user_instruction)` | Loop check | sem_filter.py:358, 363-365 |
| sem_map | `parse_cols(user_instruction)` | Loop check | sem_map.py:234, 237-239 |
| sem_join | `parse_cols(join_instruction)` + left/right resolution | Complex resolution | sem_join.py:699-730 |
| sem_agg | `parse_cols(user_instruction)` or `list(df.columns)` | Loop check | sem_agg.py:370-374, 377-379 |
| sem_topk | `parse_cols(user_instruction)` | Loop check | sem_topk.py:754, 758-760 |
| sem_extract | `input_cols` parameter (list) | Loop check | sem_extract.py:203, 222-224 |
| sem_search | `col_name` parameter (string) | Implicit via index lookup | sem_search.py:94, 111 |
| sem_sim_join | `left_on`, `right_on` parameters | Implicit via index lookup | sem_sim_join.py:88-89 |
| sem_dedup | `col_name` parameter (string) | Implicit via sim_join | sem_dedup.py:36 |
| sem_index | `col_name` parameter (string) | Implicit via `df[col_name]` | sem_index.py:62, 74 |
| sem_partition_by | Trong `partition_fn` | Delegated to fn | sem_partition_by.py:63 |
| sem_cluster_by | `col_name` parameter (string) | Implicit via utils.cluster | sem_cluster_by.py:60 |

## 4. Inconsistencies

### Error Message Format
- sem_filter, sem_map, sem_extract: `f"Column {column} not found in DataFrame"`
- sem_agg, sem_topk: `f"column {column} not found in DataFrame. Given usr instruction: {user_instruction}"`
- Khác nhau về capitalization ("Column" vs "column") và thông tin bổ sung

### Validation Approach
- LLM operators: Explicit loop check trước khi xử lý
- Embedding operators: Implicit — error xảy ra khi truy cập `df[col_name]` hoặc index lookup
- sem_partition_by: Không validate — delegate cho partition_fn

### sem_extract Exception
- Không dùng `parse_cols()` — dùng `input_cols` list
- Không dùng langex — columns được truyền explicit
- Điều này tạo inconsistency trong API: user phải nhớ 2 cách khác nhau để specify columns

### sem_agg `all_cols` Option
- Khi `all_cols=True`: `col_li = list(self._obj.columns)` (sem_agg.py:371)
- Skip `parse_cols()` hoàn toàn
- Cho phép aggregation trên tất cả columns mà không cần langex

## 5. Column Usage After Extraction

Sau khi columns được extract và validate, chúng được dùng cho:

1. **Data serialization**: `df2multimodal_info(df, col_li)` — chỉ serialize referenced columns
2. **Instruction formatting**: `nle2str(instruction, col_li)` — replace `{col}` với `Col`
3. **Example handling**: `df2multimodal_info(examples, col_li)` — serialize example data với cùng columns
4. **Join resolution**: `merge_multimodal_info()` — gộp left + right multimodal data (task_instructions.py:382-402)
