# A1 - DataFrame Extension Pattern

## Mo hinh mo rong pandas DataFrame

LOTUS su dung co che `@pd.api.extensions.register_dataframe_accessor` cua pandas de them cac semantic operator vao DataFrame. Day la pattern chinh thuc cua pandas cho phep "plugin" method moi ma khong can subclass DataFrame.

## Pattern chi tiet

Moi accessor class co cau truc giong nhau:

```python
@pd.api.extensions.register_dataframe_accessor("sem_filter")
class SemFilterDataframe:
    def __init__(self, pandas_obj):    # pandas truyen DataFrame vao day
        self._validate(pandas_obj)
        self._obj = pandas_obj         # luu tham chieu den DataFrame goc

    @staticmethod
    def _validate(obj):
        if not isinstance(obj, pd.DataFrame):
            raise AttributeError("Must be a DataFrame")

    @operator_cache                    # cache ket qua operator
    def __call__(self, user_instruction, ...):
        # Logic chinh cua operator
        ...
```

### Cach hoat dong:

1. Khi user goi `df.sem_filter(...)`, pandas tao instance `SemFilterDataframe(df)` va goi `__call__()` tren instance do.
2. `self._obj` luu tham chieu den DataFrame goc, cho phep truy cap du lieu.
3. `@operator_cache` decorator (cache.py:33) wrap `__call__`, hash `self._obj` + arguments de cache ket qua.
4. Vi du cu the: `SemFilterDataframe` tai `sem_filter.py:225-226`:
   ```python
   @pd.api.extensions.register_dataframe_accessor("sem_filter")
   class SemFilterDataframe:
   ```

## Danh sach tat ca Accessors

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

## So sanh cac loai accessor

### LLM operators
- Su dung `lotus.settings.lm` (LM instance)
- Goi `filter_formatter`, `map_formatter`, `extract_formatter` de tao prompt
- Goi `LM.__call__()` de batch completion
- Co postprocessor de parse output

### Embedding operators
- Su dung `lotus.settings.rm` (RM instance) va `lotus.settings.vs` (VS instance)
- Truy cap `self._obj.attrs["index_dirs"]` de load vector index
- Khong can LLM, chi dung embedding model

### Dac biet: `load_sem_index`
- Khong co `@operator_cache` decorator (load_sem_index.py:49)
- Chi don gian set `attrs["index_dirs"][col_name] = index_dir`
- La accessor duy nhat khong can `lotus.settings.lm` hay `lotus.settings.rm`

### Dac biet: `sem_partition_by`
- Nhan mot `partition_fn: Callable` thay vi user_instruction (sem_partition_by.py:63)
- Gan `_lotus_partition_id` column len DataFrame
- Duoc su dung truoc `sem_agg` de nhom du lieu

## Validate pattern

Hau het accessor deu validate input la DataFrame:
```python
@staticmethod
def _validate(obj):
    if not isinstance(obj, pd.DataFrame):
        raise AttributeError("Must be a DataFrame")
```

Ngoai tru `SemAggDataframe._validate` (sem_agg.py:312-323) va `SemTopKDataframe._validate` (sem_topk.py:694-705) co body rong (`pass`), khong thuc su validate.
