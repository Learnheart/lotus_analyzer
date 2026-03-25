# E6 - Large Dataset Strategies

## Tổng quan

LOTUS xử lý large datasets qua batch processing, group-by parallelism, và importance sampling. Không có explicit streaming hay partitioning framework ngoài pandas.

---

## 1. Batch Processing via LM

**Location**: `lm.py:67-73`

```python
def __init__(self, model="gpt-4o-mini", ...,
             max_batch_size=64, ...):          # :73
```

### Batch completion (`lm.py:250-253`)
```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True,
    max_workers=self.max_batch_size,           # :251
    **all_kwargs
)
```

- `max_batch_size=64` default: 64 concurrent API requests
- Dùng litellm `batch_completion` cho parallel processing
- Rate limiting adjusts batch size: `min(rate_limit, max_batch_size)` (`lm.py:108`)

### Embedding batch processing

SentenceTransformersRM (`sentence_transformers_rm.py:67-68`):
```python
for i in tqdm(range(0, len(docs), self.max_batch_size)):
    batch = docs[i : i + self.max_batch_size]
```

LiteLLMRM (`litellm_rm.py:63-64`):
```python
for i in tqdm(range(0, len(docs), self.max_batch_size)):
    batch = docs[i : i + self.max_batch_size]
```

Cả hai đều process `max_batch_size=64` documents per batch.

---

## 2. Group-by Parallelism

### sem_agg group-by

**Location**: `sem_agg.py:396-399`

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(
    max_workers=lotus.settings.parallel_groupby_max_threads
) as executor:
    return pd.concat(list(executor.map(
        SemAggDataframe.process_group, group_args
    )))
```

### sem_topk group-by

**Location**: `sem_topk.py:770-780`

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(
    max_workers=lotus.settings.parallel_groupby_max_threads
) as executor:
    results = list(executor.map(
        SemTopKDataframe.process_group, group_args
    ))
```

Mỗi group được process trong thread riêng → parallel aggregation/sorting.

---

## 3. Importance Sampling cho Cascade

**Location**: `cascade_utils.py:8-30`

```python
def importance_sampling(proxy_scores, cascade_args):
    sample_size = int(cascade_args.sampling_percentage * len(proxy_scores))  # :25
```

- Default `sampling_percentage=0.1` (`types.py:158`) → chỉ 10% data dùng để learn thresholds
- Giảm cost cho threshold learning trên large datasets
- Weighted sampling bias toward high-score items (`cascade_utils.py:16-18`)

---

## 4. Không có Explicit Streaming

LOTUS load toàn bộ data vào pandas DataFrame trước khi xử lý:
- Không có streaming/iterator-based processing
- Không có out-of-core computation
- Toàn bộ DataFrame phải fit trong memory

---

## 5. Không có Distributed Processing

Không tìm thấy:
- Spark/Dask integration
- MapReduce patterns
- Distributed LLM call scheduling
- Cross-machine parallelism

---

## 6. FAISS Scalability

FaissVS hỗ trợ approximate indices cho large datasets:

```python
class FaissVS(VS):
    def __init__(self, factory_string="Flat", ...):  # faiss_vs.py:14
```

- `"Flat"`: O(N) search, exact results
- `"IVF100,Flat"`: O(sqrt(N)) average, approximate
- `"HNSW32"`: O(log(N)) search, approximate

Nhưng default `"Flat"` không scale well cho millions of vectors.

---

## 7. Rate Limiting cho API Cost Control

### RPM (Requests Per Minute) (`lm.py:258-303`)
```python
def _process_with_rate_limiting(self, batch, all_kwargs, pbar):
    min_interval_per_request = 60 / self.rate_limit     # :279
    # ... sleep between batches ...
```

### TPM (Tokens Per Minute) (`lm.py:311-390`)
```python
def _process_with_tpm_limiting(self, batch, all_kwargs, pbar):
    available_tokens = max(0, int(self.tpm_limit * 0.95) - tokens_in_last_minute)  # :336
    # ... adaptive batch sizing ...
```

Cả hai mechanisms control throughput, không speedup nhưng prevent rate limit errors.

---

## 8. Kết luận

Large dataset handling trong LOTUS:
- **Batch processing**: max_batch_size=64 concurrent requests
- **Group-by parallelism**: ThreadPoolExecutor for parallel groups
- **Importance sampling**: 10% sample for cascade learning
- **No streaming**: All data in memory
- **No distributed**: Single-machine only
- **FAISS scalable**: Via approximate index types
- **Rate limiting**: RPM and TPM control
