# A1 - DataFrame Extension Pattern

## Mô hình mở rộng pandas DataFrame

LOTUS sử dụng cơ chế `@pd.api.extensions.register_dataframe_accessor` của pandas để thêm các semantic operator vào DataFrame. Đây là pattern chính thức của pandas cho phép "plugin" method mới mà không cần subclass DataFrame.

## Pattern chi tiết

Mỗi accessor class có cấu trúc giống nhau:

```python
@pd.api.extensions.register_dataframe_accessor("sem_filter")
class SemFilterDataframe:
    def __init__(self, pandas_obj):    # pandas truyền DataFrame vào đây
        self._validate(pandas_obj)
        self._obj = pandas_obj         # lưu tham chiếu đến DataFrame gốc

    @staticmethod
    def _validate(obj):
        if not isinstance(obj, pd.DataFrame):
            raise AttributeError("Must be a DataFrame")

    @operator_cache                    # cache kết quả operator
    def __call__(self, user_instruction, ...):
        # Logic chính của operator
        ...
```

### Cách hoạt động:

1. Khi user gọi `df.sem_filter(...)`, pandas tạo instance `SemFilterDataframe(df)` và gọi `__call__()` trên instance đó.
2. `self._obj` lưu tham chiếu đến DataFrame gốc, cho phép truy cập dữ liệu.
3. `@operator_cache` decorator (cache.py:33) wrap `__call__`, hash `self._obj` + arguments để cache kết quả.
4. Ví dụ cụ thể: `SemFilterDataframe` tại `sem_filter.py:225-226`:
   ```python
   @pd.api.extensions.register_dataframe_accessor("sem_filter")
   class SemFilterDataframe:
   ```

## Danh sách tất cả Accessors

| Accessor Name | Class | File:Line | Category |
|---|---|---|---|
| `sem_filter` | `SemFilterDataframe` | `sem_ops/sem_filter.py:225-226` | LLM operator |
| `sem_map` | `SemMapDataframe` | `sem_ops/sem_map.py:121-122` | LLM operator |
| `sem_join` | `SemJoinDataframe` | `sem_ops/sem_join.py:606-607` | LLM operator |
| `sem_agg` | `SemAggDataframe` | `sem_ops/sem_agg.py:226-227` | LLM operator |
| `sem_topk` | `SemTopKDataframe` | `sem_ops/sem_topk.py:624-625` | LLM operator |
| `sem_extract` | `SemExtractDataFrame` | `sem_ops/sem_extract.py:111-112` | LLM operator |
| `llm_as_judge` | `LLMAsJudgeDataframe` | `evals/llm_as_judge.py:117-118` | LLM eval |
| `pairwise_judge` | `PairwiseJudgeDataframe` | `evals/pairwise_judge.py:13-14` | LLM eval |
| `sem_search` | `SemSearchDataframe` | `sem_ops/sem_search.py:10-11` | Embedding operator |
| `sem_sim_join` | `SemSimJoinDataframe` | `sem_ops/sem_sim_join.py:12-13` | Embedding operator |
| `sem_dedup` | `SemDedupByDataframe` | `sem_ops/sem_dedup.py:10-11` | Embedding operator |
| `sem_index` | `SemIndexDataframe` | `sem_ops/sem_index.py:9-10` | Index management |
| `load_sem_index` | `LoadSemIndexDataframe` | `sem_ops/load_sem_index.py:6-7` | Index management |
| `sem_partition_by` | `SemPartitionByDataframe` | `sem_ops/sem_partition_by.py:8-9` | Utility |
| `sem_cluster_by` | `SemClusterByDataframe` | `sem_ops/sem_cluster_by.py:10-11` | Utility |

## So sánh các loại accessor

### LLM operators
- Sử dụng `lotus.settings.lm` (LM instance)
- Gọi `filter_formatter`, `map_formatter`, `extract_formatter` để tạo prompt
- Gọi `LM.__call__()` để batch completion
- Có postprocessor để parse output

### Embedding operators
- Sử dụng `lotus.settings.rm` (RM instance) và `lotus.settings.vs` (VS instance)
- Truy cập `self._obj.attrs["index_dirs"]` để load vector index
- Không cần LLM, chỉ dùng embedding model

### Đặc biệt: `load_sem_index`
- Không có `@operator_cache` decorator (load_sem_index.py:49)
- Chỉ đơn giản set `attrs["index_dirs"][col_name] = index_dir`
- Là accessor duy nhất không cần `lotus.settings.lm` hay `lotus.settings.rm`

### Đặc biệt: `sem_partition_by`
- Nhận một `partition_fn: Callable` thay vì user_instruction (sem_partition_by.py:63)
- Gán `_lotus_partition_id` column lên DataFrame
- Được sử dụng trước `sem_agg` để nhóm dữ liệu

## Validate pattern

Hầu hết accessor đều validate input là DataFrame:
```python
@staticmethod
def _validate(obj):
    if not isinstance(obj, pd.DataFrame):
        raise AttributeError("Must be a DataFrame")
```

Ngoại trừ `SemAggDataframe._validate` (sem_agg.py:312-323) và `SemTopKDataframe._validate` (sem_topk.py:694-705) có body rỗng (`pass`), không thực sự validate.
