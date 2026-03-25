# SEM_PARTITION_BY — `sem_partition_by`

## Metadata
- **File**: `lotus/sem_ops/sem_partition_by.py`
- **Accessor line**: 8 (`@pd.api.extensions.register_dataframe_accessor("sem_partition_by")`)
- **Type**: Unary operator
- **Requires**: Khong yeu cau model nao — chi can partition function
- **LLM**: Khong dung LLM

## 1. Purpose & Use Cases

Phan nhom DataFrame theo mot ham partition do user dinh nghia. Ket qua la column `_lotus_partition_id` duoc them vao DataFrame, su dung boi sem_agg de tong hop theo nhom.

**Use cases:**
- Cluster-based partition: `df.sem_partition_by(lotus.utils.cluster("text", 3))`
- Custom partition: `df.sem_partition_by(my_custom_partition_fn)`

## 2. Call Stack Trace

```
1. SemPartitionByDataframe.__call__()                   # sem_partition_by.py:61
2.   group_ids = partition_fn(self._obj)                # sem_partition_by.py:65
3.   df["_lotus_partition_id"] = Series(group_ids)      # sem_partition_by.py:66
4.   Return df                                          # sem_partition_by.py:67
```

## 3. Prompt Template (COPY VERBATIM)

Khong co prompt — sem_partition_by khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** Operator nay chi goi `partition_fn(df)` va luu ket qua.

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Caching | Yes | `@operator_cache` (sem_partition_by.py:60) |

## 6. Input/Output Contract

### Input:
- `partition_fn: Callable[[pd.DataFrame], list[int]]` — Ham nhan DataFrame, tra ve list partition IDs (sem_partition_by.py:63)

### Output:
- DataFrame goc + column `_lotus_partition_id` (sem_partition_by.py:66)
- Partition IDs la integers, duoc gan qua `pd.Series(group_ids, index=df.index)` de align voi DataFrame index

### Downstream usage:
- `sem_agg` kiem tra `_lotus_partition_id` column va sort/group theo no (sem_agg.py:402-406)

## 7. Edge Cases

1. **partition_fn tra ve sai length**: Pandas se raise error khi gan Series voi wrong length
2. **partition_fn raise exception**: Exception propagate len caller
3. **Da co _lotus_partition_id**: Se bi overwrite (sem_partition_by.py:66)

## 8. Code Examples

```python
# Voi lotus.utils.cluster
df = df.sem_index("text", "text_idx") \
       .sem_partition_by(lotus.utils.cluster("text", 3))

# Sau do dung voi sem_agg
df.sem_agg("Summarize {text}")

# Custom partition function
def my_partition(df):
    return [i % 3 for i in range(len(df))]

df = df.sem_partition_by(my_partition)
```

## 9. Assessment

### Diem manh:
- **Simple**: Chi 3 dong logic (sem_partition_by.py:65-67)
- **Flexible**: Bat ky partition function nao deu duoc
- **Integration**: Tuong thich tot voi sem_agg qua `_lotus_partition_id`

### Diem yeu:
- **No validation**: Khong kiem tra partition_fn output co hop le khong
- **Magic column name**: `_lotus_partition_id` la hardcoded string — coupling voi sem_agg
- **Mutates in-place**: Them column truc tiep vao `self._obj` (sem_partition_by.py:66) thay vi copy — side effect
