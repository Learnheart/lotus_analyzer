# SEM_PARTITION_BY — `sem_partition_by`

## Metadata
- **File**: `lotus/sem_ops/sem_partition_by.py`
- **Accessor line**: 8 (`@pd.api.extensions.register_dataframe_accessor("sem_partition_by")`)
- **Type**: Unary operator
- **Requires**: Không yêu cầu model nào — chỉ cần partition function
- **LLM**: Không dùng LLM

## 1. Purpose & Use Cases

Phân nhóm DataFrame theo một hàm partition do user định nghĩa. Kết quả là column `_lotus_partition_id` được thêm vào DataFrame, sử dụng bởi sem_agg để tổng hợp theo nhóm.

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

Không có prompt — sem_partition_by không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** Operator này chỉ gọi `partition_fn(df)` và lưu kết quả.

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Caching | Yes | `@operator_cache` (sem_partition_by.py:60) |

## 6. Input/Output Contract

### Input:
- `partition_fn: Callable[[pd.DataFrame], list[int]]` — Hàm nhận DataFrame, trả về list partition IDs (sem_partition_by.py:63)

### Output:
- DataFrame gốc + column `_lotus_partition_id` (sem_partition_by.py:66)
- Partition IDs là integers, được gán qua `pd.Series(group_ids, index=df.index)` để align với DataFrame index

### Downstream usage:
- `sem_agg` kiểm tra `_lotus_partition_id` column và sort/group theo nó (sem_agg.py:402-406)

## 7. Edge Cases

1. **partition_fn trả về sai length**: Pandas sẽ raise error khi gán Series với wrong length
2. **partition_fn raise exception**: Exception propagate lên caller
3. **Đã có _lotus_partition_id**: Sẽ bị overwrite (sem_partition_by.py:66)

## 8. Code Examples

```python
# Với lotus.utils.cluster
df = df.sem_index("text", "text_idx") \
       .sem_partition_by(lotus.utils.cluster("text", 3))

# Sau đó dùng với sem_agg
df.sem_agg("Summarize {text}")

# Custom partition function
def my_partition(df):
    return [i % 3 for i in range(len(df))]

df = df.sem_partition_by(my_partition)
```

## 9. Assessment

### Điểm mạnh:
- **Simple**: Chỉ 3 dòng logic (sem_partition_by.py:65-67)
- **Flexible**: Bất kỳ partition function nào đều được
- **Integration**: Tương thích tốt với sem_agg qua `_lotus_partition_id`

### Điểm yếu:
- **No validation**: Không kiểm tra partition_fn output có hợp lệ không
- **Magic column name**: `_lotus_partition_id` là hardcoded string — coupling với sem_agg
- **Mutates in-place**: Thêm column trực tiếp vào `self._obj` (sem_partition_by.py:66) thay vì copy — side effect
