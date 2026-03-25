# C0 - Tổng quan các kỹ thuật Optimization trong LOTUS

## Mục tiêu

LOTUS sử dụng nhiều kỹ thuật optimization để giảm chi phí LLM calls, tăng throughput, và đảm bảo chất lượng.
Tài liệu này tóm tắt toàn bộ các kỹ thuật optimization có trong codebase.

---

## 1. Model Cascading (Small -> Large Model Routing)

**Nguyên lý:** Dùng model rẻ (proxy) trước, chỉ gửi các trường hợp không chắc chắn đến model đắt (oracle).

**Các operator hỗ trợ:**
- `sem_filter` cascade: `sem_filter.py:383-530` — dùng helper_lm logprobs hoặc embedding similarity làm proxy scores
- `sem_join` cascade: `sem_join.py:180-333` — join_optimizer so sánh Search-Filter vs Map-Search-Filter
- `sem_topk` cascade: `sem_topk.py:176-273` — `compare_batch_binary_cascade` dùng helper_lm với logprobs

**Cấu hình:** `CascadeArgs` tại `types.py:155-176` bao gồm `recall_target`, `precision_target`, `sampling_percentage`, `failure_probability`.

**Proxy models:** `ProxyModel` enum tại `types.py:150-152` — `HELPER_LM` hoặc `EMBEDDING_MODEL`.

---

## 2. Operator-Level Caching (@operator_cache)

**Vị trí:** Decorator `operator_cache` tại `cache.py:33-100`.

**Cách hoạt động:**
- Key = sha256(self._obj + args + kwargs serialized) — `cache.py:72-76`
- Cache toàn bộ kết quả operator (DataFrame)
- Cũng cache `virtual_usage` để đảm bảo thống kê chính xác — `cache.py:77`, `cache.py:84-86`
- Mọi operator đều được decorator này: `sem_filter.py:333`, `sem_map.py:214`, `sem_topk.py:734`, `sem_agg.py:353`, `sem_join.py:669`, v.v.

**Bật/Tắt:** `lotus.settings.enable_cache` (default `False`) — `settings.py:17`

---

## 3. LM-Level Caching (Response Cache)

**Vị trí:** `lm.py:136-190` trong phương thức `__call__`.

**Cách hoạt động:**
- Key = sha256(model + messages + kwargs) — `lm.py:407-410`
- Trước khi gọi LLM, check cache để lấy cached responses — `lm.py:138-149`
- Responses uncached được xử lý và lưu vào cache — `lm.py:168-171`

**Các implementation:**
- `InMemoryCache`: OrderedDict LRU eviction — `cache.py:247-268`
- `SQLiteCache`: SQLite-backed persistent cache — `cache.py:168-244`
- Default: `InMemoryCache(max_size=1024)` — `cache.py:146-147`

---

## 4. Batching (litellm batch_completion)

**LM batching:**
- Default `max_batch_size=64` — `lm.py:73`
- Sử dụng `litellm.batch_completion` với `max_workers=max_batch_size` — `lm.py:250-252`
- Rate-limited batching: chia sub-batches + sleep — `lm.py:258-303`
- TPM-limited batching: ước lượng token budget mỗi batch — `lm.py:311-390`

**Embedding batching:**
- `SentenceTransformersRM`: batch với `max_batch_size=64` — `sentence_transformers_rm.py:29`
- `LiteLLMRM`: batch với `max_batch_size=64` — `litellm_rm.py:28`

---

## 5. Rate/TPM Limiting

**Rate limiting (RPM):**
- `rate_limit` parameter trên LM constructor — `lm.py:74`
- Delay = `60 / rate_limit` giữa các batch — `lm.py:106`
- `max_batch_size` được cấp bởi `rate_limit` — `lm.py:108`
- Implementation: `_process_with_rate_limiting` — `lm.py:258-303`

**TPM limiting:**
- `tpm_limit` parameter — `lm.py:75`
- 95% safety buffer — `lm.py:317`
- Token usage history tracking với 60-second window — `lm.py:305-309`
- Per-request token estimation qua `count_tokens` — `lm.py:319-321`
- Chờ khi token window giải phóng nếu hết budget — `lm.py:380-388`

---

## 6. Join Optimization (Search-Filter vs Map-Search-Filter)

**Vị trí:** `join_optimizer` tại `sem_join.py:417-527`.

**Đây là query planner DUY NHẤT thực sự trong LOTUS.**

**Hai strategies:**
1. **Search-Filter:** sem_sim_join trực tiếp — `sem_join.py:463`
2. **Map-Search-Filter:** map l1 trước (sem_map), rồi sem_sim_join — `sem_join.py:483-486`

**Cách chọn:**
- Tính cascade thresholds cho cả hai strategies
- Đếm số LLM calls cần thiết cho low-confidence items
- Chọn plan có ít LLM calls hơn — `sem_join.py:518-527`

---

## 7. Embedding-Based Pre-filtering cho TopK

**Vị trí:** `sem_topk.py:782-788` (method `quick-sem`).

**Cách hoạt động:**
- Trước khi sorting, dùng `sem_index` + `sem_search` để pre-sort DataFrame theo embedding similarity
- Pivot selection dùng `heapq.nsmallest` thay vì random — `sem_topk.py:411-417`
- Giảm số LLM comparison calls vì data đã được pre-sorted

---

## Bảng tóm tắt

| Kỹ thuật | Áp dụng cho | Mục tiêu chính | File chính |
|----------|-------------|----------------|------------|
| Model Cascading | filter, join, topk | Giảm LLM calls 50-90% | cascade_utils.py, sem_filter.py, sem_join.py, sem_topk.py |
| Operator Cache | Tất cả operators | Tránh tính toán lặp lại | cache.py:33 |
| LM Response Cache | Mọi LLM call | Tránh duplicate API calls | lm.py:136, cache.py:115 |
| Batch Processing | LM, RM | Tăng throughput | lm.py:250, sentence_transformers_rm.py:67 |
| Rate/TPM Limiting | LM | Tương thích API limits | lm.py:258, lm.py:311 |
| Join Optimizer | sem_join | Chọn strategy rẻ nhất | sem_join.py:417 |
| Embedding Pre-filter | sem_topk | Giảm LLM comparisons | sem_topk.py:782 |
