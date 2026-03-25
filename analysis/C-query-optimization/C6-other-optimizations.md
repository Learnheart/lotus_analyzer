# C6 - Các Optimization khác trong LOTUS

## 1. safe_mode — Ước lượng Chi phí trước khi Thực thi

### Mục đích
Cho phép user xem ước lượng token/cost trước khi chạy operator, tránh bất ngờ về chi phí.

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

**KHÔNG được implement:**
- `sem_agg` — `sem_agg.py:152-153`: `"Safe mode is not implemented yet"`
- `sem_join` cascade — `sem_join.py:262-264`: `"Safe mode is not implemented yet"`

---

## 2. Embedding-Based Pivot Selection trong Quicksort

### Vị trí
`sem_topk.py:411-417` trong hàm `partition`.

### Cách hoạt động
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

**Giải thích:**
- Khi `embedding=True` (method "quick-sem"), DataFrame đã được pre-sorted theo embedding similarity — `sem_topk.py:782-788`
- `indexes` là mapping từ sorted position về original index
- `heapq.nsmallest(K, indexes)` chọn item có index nhỏ nhất (= ranking cao nhất theo embedding)
- Dùng item này làm pivot → partition sẽ chia data gần đúng tại vị trí K
- Giảm số recursive calls cần thiết

**Kích hoạt:** `df.sem_topk("...", K=5, method="quick-sem")` — `sem_topk.py:782-788`

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

### Vị trí
`sem_search.py:127-138`

### Vấn đề
Khi DataFrame đã được filter (chỉ còn một subset của rows), vector index vẫn chứa tất cả rows gốc.
Vector search có thể trả về rows đã bị filter ra.

### Giải pháp
```python
# sem_search.py:116-138
df_idxs = self._obj.index  # Các index hiện tại của DataFrame
search_K = K
while True:
    vs_output = vs(query_vectors, search_K)
    doc_idxs = vs_output.indices[0]
    scores = vs_output.distances[0]

    # Post-filter: chỉ giữ các results có trong DataFrame hiện tại
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
    search_K = search_K * 2  # Tăng K và thử lại
```

**Key insight:** Nếu không đủ K results sau post-filter, tăng `search_K` gấp đôi và chạy lại.
Đây là cách đơn giản để đảm bảo luôn trả về đủ K results từ filtered DataFrame.

---

## 4. Parallel GroupBy trong sem_agg và sem_topk

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

**Cấu hình:** `settings.py:23` — `parallel_groupby_max_threads: int = 8`

**Lưu ý:**
- Python GIL giới hạn CPU parallelism, nhưng LLM calls là I/O-bound nên threads vẫn hiệu quả
- Mỗi group chạy độc lập với model riêng của nó
- Kết quả được `pd.concat` lại — `sem_agg.py:399`, `sem_topk.py:776-780`

---

## 5. Hierarchical Aggregation Tree

### Vị trí
`sem_agg.py:60-223`

### Cách hoạt động
Khi có nhiều documents hơn context window của model, sem_agg xây dựng cây aggregation:

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

**Template khác nhau cho mỗi level:**
- Leaf template — `sem_agg.py:12-31`: "given the context below from multiple documents"
- Node template — `sem_agg.py:34-57`: "given the context below from multiple sources"

**Partition-aware:** Documents có cùng `partition_id` sẽ được aggregation với nhau trước — `sem_agg.py:179-184`.

---

## 6. Long Context Strategy cho Aggregation

### Vị trí
`long_context_strategy.py:1-50`, sử dụng trong `sem_agg.py:411-428`

### Hai strategies

**TRUNCATE:** Cắt documents khi quá dài — đơn giản nhưng mất thông tin.

**CHUNK:** Chia document lớn thành các chunks nhỏ hơn — `long_context_strategy.py:22-50`
- `ChunkedDocument` dataclass lưu trữ chunks + metadata
- `chunk_info` ghi lại original row index và chunk index — `long_context_strategy.py:12-18`
- Cho phép khôi phục lại row gốc sau khi xử lý

### Sử dụng
```python
# sem_agg.py:418-419
docs_input = create_chunked_documents(
    self._obj, col_li, lotus.settings.lm, long_context_strategy, template_tokens
)
```
Default: `LongContextStrategy.CHUNK` — `sem_agg.py:362`

---

## 7. Usage Limits

### Vị trí
`types.py:215-220` và `lm.py:419-427`

```python
@dataclass
class UsageLimit:
    prompt_tokens_limit: float = float("inf")
    completion_tokens_limit: float = float("inf")
    total_tokens_limit: float = float("inf")
    total_cost_limit: float = float("inf")
```

### Cách hoạt động
```python
# lm.py:419-427
def _check_usage_limit(self, usage, limit, usage_type):
    if (usage.prompt_tokens > limit.prompt_tokens_limit
        or usage.completion_tokens > limit.completion_tokens_limit
        or usage.total_tokens > limit.total_tokens_limit
        or usage.total_cost > limit.total_cost_limit):
        raise LotusUsageLimitException(...)
```

- Kiểm tra sau mỗi response — `lm.py:478`, `lm.py:483`
- Có cả `physical_usage_limit` và `virtual_usage_limit` — `lm.py:78-79`
- Raise `LotusUsageLimitException` khi vượt limit — `types.py:232-234`

---

## 8. Tóm tắt

| Optimization | Mục đích | Vị trí | Trạng thái |
|-------------|---------|--------|-----------|
| safe_mode | Ước lượng chi phí | Nhiều operators | Một phần (chưa đầy đủ) |
| Embedding pivot | Giảm LLM calls trong quicksort | sem_topk.py:411-417 | Hoàn chỉnh |
| Post-filtering | Xử lý filtered DataFrames | sem_search.py:127-138 | Hoàn chỉnh |
| Parallel groupby | Tăng throughput | sem_agg.py:396, sem_topk.py:772 | Hoàn chỉnh |
| Hierarchical aggregation | Xử lý data lớn hơn context window | sem_agg.py:164-219 | Hoàn chỉnh |
| Long context strategy | Xử lý documents quá dài | long_context_strategy.py | Hoàn chỉnh |
| Usage limits | Tránh chi phí quá cao | lm.py:419-427 | Hoàn chỉnh |
