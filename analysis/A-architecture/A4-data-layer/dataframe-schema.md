# A4 - DataFrame Schema

## DataFrame Metadata: attrs

LOTUS sử dụng `DataFrame.attrs` (pandas built-in metadata dict) để lưu thông tin về vector indices.

### attrs["index_dirs"]

```python
# Được set bởi sem_index (sem_index.py:76):
self._obj.attrs["index_dirs"][col_name] = index_dir

# Được set bởi load_sem_index (load_sem_index.py:50):
self._obj.attrs["index_dirs"][col_name] = index_dir
```

Format: `dict[str, str]` - mapping từ column name -> index directory path.

Ví dụ:
```python
df.attrs["index_dirs"] = {
    "title": "title_index",
    "content": "content_index"
}
```

### Propagation qua operations:

```python
# sem_filter.py:564
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)

# sem_search.py:141
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)
```

### Cảnh báo: Reset trong __init__

Cả `SemIndexDataframe.__init__` (sem_index.py:54) và `LoadSemIndexDataframe.__init__` (load_sem_index.py:42) đều reset `attrs["index_dirs"] = {}`. Điều này có thể mất index references khi accessor được truy cập nhiều lần.

---

## Special Columns

### _lotus_partition_id

Được tạo bởi `sem_partition_by` (sem_partition_by.py:66):
```python
self._obj["_lotus_partition_id"] = pd.Series(group_ids, index=self._obj.index)
```

Được đọc bởi `sem_agg` (sem_agg.py:402-406):
```python
if "_lotus_partition_id" in self._obj.columns:
    self._obj = self._obj.sort_values(by="_lotus_partition_id")
    partition_ids = self._obj["_lotus_partition_id"].tolist()
else:
    partition_ids = [0] * len(self._obj)
```

Mục đích: Group documents trước khi aggregation. Documents có cùng `_lotus_partition_id` được aggregate cùng nhau, tạo ra 1 summary cho mỗi partition.

### cluster_id

Được tạo bởi `sem_cluster_by` (sem_cluster_by.py:78):
```python
self._obj["cluster_id"] = pd.Series(indices, index=self._obj.index)
```

Kết quả của faiss k-means clustering trên embedding vectors.

### _scores

Được tạo bởi `sem_sim_join` (sem_sim_join.py:151):
```python
temp_df = pd.DataFrame(join_results, columns=["_left_id", "_right_id", "_scores" + score_suffix])
```

Similarity scores từ vector search, dùng trong join cascade threshold learning.

### vec_scores{suffix}

Được tạo bởi `sem_search` khi `return_scores=True` (sem_search.py:144):
```python
new_df["vec_scores" + suffix] = postfiltered_scores
```

Default suffix là `"_sim_score"` -> column name `"vec_scores_sim_score"`.

### filter_label

Được tạo bởi `sem_filter` khi `return_all=True` (sem_filter.py:577):
```python
new_df[get_out_col_name(new_df, "filter_label")] = outputs
```

Boolean values (True/False) cho mỗi row.

### {suffix} columns

Mỗi operator thêm columns với suffix:
- `sem_map`: `"_map"` (sem_map.py:222)
- `sem_filter`: `"_filter"` (sem_filter.py:341)
- `sem_join`: `"_join"` (sem_join.py:672)
- `sem_agg`: `"_output"` (sem_agg.py:358)
- `sem_extract`: Columns theo `output_cols` keys
- `llm_as_judge`: `"_judge"` (llm_as_judge.py:197)

Optional columns:
- `"explanation{suffix}"`: khi `return_explanations=True`
- `"raw_output{suffix}"`: khi `return_raw_outputs=True`

---

## Column Types

### Standard pandas types
- `str` (object): Tất cả text columns
- `int64`, `float64`: Numeric columns
- `bool`: Filter labels

### ImageDtype Extension

Định nghĩa tại `dtype_extensions/image.py` (referenced in task_instructions.py:7):
```python
from lotus.dtype_extensions import ImageDtype
```

Detection trong `df2multimodal_info` (task_instructions.py:369):
```python
image_cols = [col for col in cols if isinstance(df[col].dtype, ImageDtype)]
text_cols = [col for col in cols if col not in image_cols]
```

Image data được convert sang base64 cho multimodal LLM input (task_instructions.py:375):
```python
"image": {col.capitalize(): df[col].array.get_image(i, "base64") for col in image_cols}
```

---

## Join Column Naming

Khi 2 DataFrames có cùng column names trong `sem_join`, suffixes được thêm (sem_join.py:803-806):
```python
for col in df1.columns:
    if col in df2.columns:
        df1.rename(columns={col: col + ":left"}, inplace=True)
        df2.rename(columns={col: col + ":right"}, inplace=True)
```

User có thể sử dụng `:left` và `:right` suffixes trong join instruction (sem_join.py:703-707):
```python
if ":left" in col:
    left_on = col
    real_left_on = col.split(":left")[0]
```
