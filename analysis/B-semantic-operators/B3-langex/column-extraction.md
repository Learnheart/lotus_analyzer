# B3 — Column Extraction & Validation

> **Phan tich** cach columns duoc trich xuat tu langex va validated trong moi operator.

## 1. Extraction Pipeline

### Giai doan 1: Parse columns tu langex

```python
# nl_expression.py:4-14
col_li = lotus.nl_expression.parse_cols(user_instruction)
```

Regex `r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"` trich xuat tat ca `{col}` patterns.

**Input**: `"The {text} about {topic} is positive"`
**Output**: `["text", "topic"]`

### Giai doan 2: Validate columns ton tai trong DataFrame

Moi operator co validation loop rieng:

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
Luu y: sem_agg co error message khac — them instruction vao message.

**sem_topk** (sem_topk.py:758-760):
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"column {column} not found in DataFrame. Given usr instruction: {user_instruction}")
```

**sem_extract** (sem_extract.py:222-224):
```python
for column in input_cols:  # Khong dung parse_cols!
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

### Giai doan 3: Data serialization

```python
# task_instructions.py:364-379
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
```

Chi serialize cac columns duoc reference — khong serialize toan bo DataFrame.

## 2. Special Cases: sem_join Column Resolution

sem_join co logic phuc tap hon de resolve columns giua 2 DataFrames (sem_join.py:699-730):

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
1. Neu co `:left`/`:right` suffix → dung truc tiep
2. Neu khong → tim column trong left DF truoc, roi right DF
3. Neu column ton tai trong ca 2 DFs → Raise `ValueError`

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
- Khac nhau ve capitalization ("Column" vs "column") va thong tin bo sung

### Validation Approach
- LLM operators: Explicit loop check truoc khi xu ly
- Embedding operators: Implicit — error xay ra khi truy cap `df[col_name]` hoac index lookup
- sem_partition_by: Khong validate — delegate cho partition_fn

### sem_extract Exception
- Khong dung `parse_cols()` — dung `input_cols` list
- Khong dung langex — columns duoc truyen explicit
- Dieu nay tao inconsistency trong API: user phai nho 2 cach khac nhau de specify columns

### sem_agg `all_cols` Option
- Khi `all_cols=True`: `col_li = list(self._obj.columns)` (sem_agg.py:371)
- Skip `parse_cols()` hoan toan
- Cho phep aggregation tren tat ca columns ma khong can langex

## 5. Column Usage After Extraction

Sau khi columns duoc extract va validate, chung duoc dung cho:

1. **Data serialization**: `df2multimodal_info(df, col_li)` — chi serialize referenced columns
2. **Instruction formatting**: `nle2str(instruction, col_li)` — replace `{col}` voi `Col`
3. **Example handling**: `df2multimodal_info(examples, col_li)` — serialize example data voi cung columns
4. **Join resolution**: `merge_multimodal_info()` — gop left + right multimodal data (task_instructions.py:382-402)
