# C6 - Cac Optimization khac trong LOTUS

## 1. safe_mode — Uoc luong Chi phi truoc khi Thuc thi

### Muc dich
Cho phep user xem uoc luong token/cost truoc khi chay operator, tranh bat ngo ve chi phi.

### Implementation

**sem_filter** — `sem_filter.py:107-110`:
```python
if safe_mode:
    estimated_total_calls = len(docs)
    estimated_total_cost = sum(model.count_tokens(input) for input in inputs)
    show_safe_mode(estimated_total_cost, estimated_total_calls)
```

**sem_join** — `sem_join.py:104-120`:
```python
if safe_mode:
    estimated_tokens_per_call = model.count_tokens(sample_prompt)
    estimated_total_calls = len(l1) * len(l2)
    estimated_total_cost = estimated_tokens_per_call * estimated_total_calls
    show_safe_mode(estimated_total_cost, estimated_total_calls)
```

**sem_topk quicksort** — `sem_topk.py:393-399`:
```python
if safe_mode:
    estimated_quickselect_calls = 2 * K
    estimated_quicksort_calls = 2 * len(docs) * np.log(len(docs))
    estimated_total_calls = estimated_quickselect_calls + estimated_quicksort_calls
    estimated_total_tokens = model.count_tokens(sample_prompt) * estimated_total_calls
    show_safe_mode(estimated_total_tokens, estimated_total_calls)
```

**sem_topk heapsort** — `sem_topk.py:597-603`:
```python
if safe_mode:
    estimated_heap_construction_calls = len(docs) * np.log(len(docs))
    estimated_top_k_extraction_calls = K * np.log(len(docs))
    estimated_total_calls = estimated_heap_construction_calls + estimated_top_k_extraction_calls
```

**sem_map** — `sem_map.py:96-99`:
```python
if safe_mode:
    estimated_cost = sum(model.count_tokens(input) for input in inputs)
    estimated_LM_calls = len(docs)
    show_safe_mode(estimated_cost, estimated_LM_calls)
```

**KHONG duoc implement:**
- `sem_agg` — `sem_agg.py:152-153`: `"Safe mode is not implemented yet"`
- `sem_join` cascade — `sem_join.py:262-264`: `"Safe mode is not implemented yet"`

---

## 2. Embedding-Based Pivot Selection trong Quicksort

### Vi tri
`sem_topk.py:411-417` trong ham `partition`.

### Cach hoat dong
```python
if embedding:
    if K <= high - low:
        pivot_value = heapq.nsmallest(K, indexes[low : high + 1])[-1]
    else:
        pivot_value = heapq.nsmallest(int((high - low + 1) / 2), indexes[low : high + 1])[-1]
    pivot_index = indexes.index(pivot_value)
else:
    pivot_index = np.random.randint(low, high + 1)
```

**Giai thich:**
- Khi `embedding=True` (method "quick-sem"), DataFrame da duoc pre-sorted theo embedding similarity — `sem_topk.py:782-788`
- `indexes` la mapping tu sorted position ve original index
- `heapq.nsmallest(K, indexes)` chon item co index nho nhat (= ranking cao nhat theo embedding)
- Dung item nay lam pivot → partition se chia data gan dung tai vi tri K
- Giam so recursive calls can thiet

**Kich hoat:** `df.sem_topk("...", K=5, method="quick-sem")` — `sem_topk.py:782-788`

Pre-sorting:
```python
# sem_topk.py:783-788
if method == "quick-sem":
    assert len(col_li) == 1
    col_name = col_li[0]
    self._obj = self._obj.sem_index(col_name, f"{col_name}_lotus_index") \
                         .sem_search(col_name, user_instruction, len(self._obj))
```

---

## 3. Post-filtering trong sem_search

### Vi tri
`sem_search.py:127-138`

### Van de
Khi DataFrame da duoc filter (chi con mot subset cua rows), vector index van chua tat ca rows goc.
Vector search co the tra ve rows da bi filter ra.

### Giai phap
```python
# sem_search.py:116-138
df_idxs = self._obj.index  # Cac index hien tai cua DataFrame
search_K = K
while True:
    vs_output = vs(query_vectors, search_K)
    doc_idxs = vs_output.indices[0]
    scores = vs_output.distances[0]

    # Post-filter: chi giu cac results co trong DataFrame hien tai
    postfiltered_doc_idxs = []
    postfiltered_scores = []
    for idx, score in zip(doc_idxs, scores):
        if idx in df_idxs:
            postfiltered_doc_idxs.append(idx)
            postfiltered_scores.append(score)

    postfiltered_doc_idxs = postfiltered_doc_idxs[:K]
    postfiltered_scores = postfiltered_scores[:K]
    if len(postfiltered_doc_idxs) == K:
        break
    search_K = search_K * 2  # Tang K va thu lai
```

**Key insight:** Neu khong du K results sau post-filter, tang `search_K` gap doi va chay lai.
Day la cach don gian de dam bao luon tra ve du K results tu filtered DataFrame.

---

## 4. Parallel GroupBy trong sem_agg va sem_topk

### sem_agg — `sem_agg.py:381-399`
```python
if group_by:
    grouped = self._obj.groupby(group_by)
    group_args = [(group_name, group, user_instruction, all_cols, group_by, suffix,
                   progress_bar_desc, long_context_strategy) for group_name, group in grouped]
    from concurrent.futures import ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
        return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

### sem_topk — `sem_topk.py:763-780`
```python
if group_by:
    grouped = self._obj.groupby(group_by)
    group_args = [(group, user_instruction, K, method, strategy, None,
                   cascade_threshold, return_stats) for _, group in grouped]
    from concurrent.futures import ThreadPoolExecutor
    with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
        results = list(executor.map(SemTopKDataframe.process_group, group_args))
```

**Cau hinh:** `settings.py:23` — `parallel_groupby_max_threads: int = 8`

**Luu y:**
- Python GIL gioi han CPU parallelism, nhung LLM calls la I/O-bound nen threads van hieu qua
- Moi group chay doc lap voi model rieng cua no
- Ket qua duoc `pd.concat` lai — `sem_agg.py:399`, `sem_topk.py:776-780`

---

## 5. Hierarchical Aggregation Tree

### Vi tri
`sem_agg.py:60-223`

### Cach hoat dong
Khi co nhieu documents hon context window cua model, sem_agg xay dung cay aggregation:

```
Level 0 (Leaf):    [doc1, doc2, doc3] → summary_A    [doc4, doc5, doc6] → summary_B
Level 1 (Node):    [summary_A, summary_B] → final_summary
```

**Token budget management** — `sem_agg.py:183-184`:
```python
if (new_tokens + context_tokens + template_tokens > model.max_ctx_len - model.max_tokens) or \
   (partition_id != cur_partition_id and not do_fold):
    # Close current prompt, start new one
```

**Template khac nhau cho moi level:**
- Leaf template — `sem_agg.py:12-31`: "given the context below from multiple documents"
- Node template — `sem_agg.py:34-57`: "given the context below from multiple sources"

**Partition-aware:** Documents co cung `partition_id` se duoc aggregation voi nhau truoc — `sem_agg.py:179-184`.

---

## 6. Long Context Strategy cho Aggregation

### Vi tri
`long_context_strategy.py:1-50`, su dung trong `sem_agg.py:411-428`

### Hai strategies

**TRUNCATE:** Cat documents khi qua dai — don gian nhung mat thong tin.

**CHUNK:** Chia document lon thanh cac chunks nho hon — `long_context_strategy.py:22-50`
- `ChunkedDocument` dataclass luu tru chunks + metadata
- `chunk_info` ghi lai original row index va chunk index — `long_context_strategy.py:12-18`
- Cho phep khoi phuc lai row goc sau khi xu ly

### Su dung
```python
# sem_agg.py:418-419
docs_input = create_chunked_documents(
    self._obj, col_li, lotus.settings.lm, long_context_strategy, template_tokens
)
```
Default: `LongContextStrategy.CHUNK` — `sem_agg.py:362`

---

## 7. Usage Limits

### Vi tri
`types.py:215-220` va `lm.py:419-427`

```python
@dataclass
class UsageLimit:
    prompt_tokens_limit: float = float("inf")
    completion_tokens_limit: float = float("inf")
    total_tokens_limit: float = float("inf")
    total_cost_limit: float = float("inf")
```

### Cach hoat dong
```python
# lm.py:419-427
def _check_usage_limit(self, usage, limit, usage_type):
    if (usage.prompt_tokens > limit.prompt_tokens_limit
        or usage.completion_tokens > limit.completion_tokens_limit
        or usage.total_tokens > limit.total_tokens_limit
        or usage.total_cost > limit.total_cost_limit):
        raise LotusUsageLimitException(...)
```

- Kiem tra sau moi response — `lm.py:478`, `lm.py:483`
- Co ca `physical_usage_limit` va `virtual_usage_limit` — `lm.py:78-79`
- Raise `LotusUsageLimitException` khi vuot limit — `types.py:232-234`

---

## 8. Tom tat

| Optimization | Muc dich | Vi tri | Trang thai |
|-------------|---------|--------|-----------|
| safe_mode | Uoc luong chi phi | Nhieu operators | Mot phan (chua day du) |
| Embedding pivot | Giam LLM calls trong quicksort | sem_topk.py:411-417 | Hoan chinh |
| Post-filtering | Xu ly filtered DataFrames | sem_search.py:127-138 | Hoan chinh |
| Parallel groupby | Tang throughput | sem_agg.py:396, sem_topk.py:772 | Hoan chinh |
| Hierarchical aggregation | Xu ly data lon hon context window | sem_agg.py:164-219 | Hoan chinh |
| Long context strategy | Xu ly documents qua dai | long_context_strategy.py | Hoan chinh |
| Usage limits | Tranh chi phi qua cao | lm.py:419-427 | Hoan chinh |
