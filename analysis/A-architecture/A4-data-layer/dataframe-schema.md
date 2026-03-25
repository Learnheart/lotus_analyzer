# A4 - DataFrame Schema

## DataFrame Metadata: attrs

LOTUS su dung `DataFrame.attrs` (pandas built-in metadata dict) de luu thong tin ve vector indices.

### attrs["index_dirs"]

```python
# Duoc set boi sem_index (sem_index.py:76):
self._obj.attrs["index_dirs"][col_name] = index_dir

# Duoc set boi load_sem_index (load_sem_index.py:50):
self._obj.attrs["index_dirs"][col_name] = index_dir
```

Format: `dict[str, str]` - mapping tu column name -> index directory path.

Vi du:
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

### Canh bao: Reset trong __init__

Ca `SemIndexDataframe.__init__` (sem_index.py:54) va `LoadSemIndexDataframe.__init__` (load_sem_index.py:42) deu reset `attrs["index_dirs"] = {}`. Dieu nay co the mat index references khi accessor duoc truy cap nhieu lan.

---

## Special Columns

### _lotus_partition_id

Duoc tao boi `sem_partition_by` (sem_partition_by.py:66):
```python
self._obj["_lotus_partition_id"] = pd.Series(group_ids, index=self._obj.index)
```

Duoc doc boi `sem_agg` (sem_agg.py:402-406):
```python
if "_lotus_partition_id" in self._obj.columns:
    self._obj = self._obj.sort_values(by="_lotus_partition_id")
    partition_ids = self._obj["_lotus_partition_id"].tolist()
else:
    partition_ids = [0] * len(self._obj)
```

Muc dich: Group documents truoc khi aggregation. Documents co cung `_lotus_partition_id` duoc aggregate cung nhau, tao ra 1 summary cho moi partition.

### cluster_id

Duoc tao boi `sem_cluster_by` (sem_cluster_by.py:78):
```python
self._obj["cluster_id"] = pd.Series(indices, index=self._obj.index)
```

Ket qua cua faiss k-means clustering tren embedding vectors.

### _scores

Duoc tao boi `sem_sim_join` (sem_sim_join.py:151):
```python
temp_df = pd.DataFrame(join_results, columns=["_left_id", "_right_id", "_scores" + score_suffix])
```

Similarity scores tu vector search, dung trong join cascade threshold learning.

### vec_scores{suffix}

Duoc tao boi `sem_search` khi `return_scores=True` (sem_search.py:144):
```python
new_df["vec_scores" + suffix] = postfiltered_scores
```

Default suffix la `"_sim_score"` -> column name `"vec_scores_sim_score"`.

### filter_label

Duoc tao boi `sem_filter` khi `return_all=True` (sem_filter.py:577):
```python
new_df[get_out_col_name(new_df, "filter_label")] = outputs
```

Boolean values (True/False) cho moi row.

### {suffix} columns

Moi operator them columns voi suffix:
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
- `str` (object): Tat ca text columns
- `int64`, `float64`: Numeric columns
- `bool`: Filter labels

### ImageDtype Extension

Dinh nghia tai `dtype_extensions/image.py` (referenced in task_instructions.py:7):
```python
from lotus.dtype_extensions import ImageDtype
```

Detection trong `df2multimodal_info` (task_instructions.py:369):
```python
image_cols = [col for col in cols if isinstance(df[col].dtype, ImageDtype)]
text_cols = [col for col in cols if col not in image_cols]
```

Image data duoc convert sang base64 cho multimodal LLM input (task_instructions.py:375):
```python
"image": {col.capitalize(): df[col].array.get_image(i, "base64") for col in image_cols}
```

---

## Join Column Naming

Khi 2 DataFrames co cung column names trong `sem_join`, suffixes duoc them (sem_join.py:803-806):
```python
for col in df1.columns:
    if col in df2.columns:
        df1.rename(columns={col: col + ":left"}, inplace=True)
        df2.rename(columns={col: col + ":right"}, inplace=True)
```

User co the su dung `:left` va `:right` suffixes trong join instruction (sem_join.py:703-707):
```python
if ":left" in col:
    left_on = col
    real_left_on = col.split(":left")[0]
```
