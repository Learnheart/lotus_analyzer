# C0 - Tong quan cac ky thuat Optimization trong LOTUS

## Muc tieu

LOTUS su dung nhieu ky thuat optimization de giam chi phi LLM calls, tang throughput, va dam bao chat luong.
Tai lieu nay tom tat toan bo cac ky thuat optimization co trong codebase.

---

## 1. Model Cascading (Small -> Large Model Routing)

**Nguyen ly:** Dung model re (proxy) truoc, chi gui cac truong hop khong chac chan den model dat (oracle).

**Cac operator ho tro:**
- `sem_filter` cascade: `sem_filter.py:383-530` — dung helper_lm logprobs hoac embedding similarity lam proxy scores
- `sem_join` cascade: `sem_join.py:180-333` — join_optimizer so sanh Search-Filter vs Map-Search-Filter
- `sem_topk` cascade: `sem_topk.py:176-273` — `compare_batch_binary_cascade` dung helper_lm voi logprobs

**Cau hinh:** `CascadeArgs` tai `types.py:155-176` bao gom `recall_target`, `precision_target`, `sampling_percentage`, `failure_probability`.

**Proxy models:** `ProxyModel` enum tai `types.py:150-152` — `HELPER_LM` hoac `EMBEDDING_MODEL`.

---

## 2. Operator-Level Caching (@operator_cache)

**Vi tri:** Decorator `operator_cache` tai `cache.py:33-100`.

**Cach hoat dong:**
- Key = sha256(self._obj + args + kwargs serialized) — `cache.py:72-76`
- Cache toan bo ket qua operator (DataFrame)
- Cung cache `virtual_usage` de dam bao thong ke chinh xac — `cache.py:77`, `cache.py:84-86`
- Moi operator deu duoc decorator nay: `sem_filter.py:333`, `sem_map.py:214`, `sem_topk.py:734`, `sem_agg.py:353`, `sem_join.py:669`, v.v.

**Bat/Tat:** `lotus.settings.enable_cache` (default `False`) — `settings.py:17`

---

## 3. LM-Level Caching (Response Cache)

**Vi tri:** `lm.py:136-190` trong phuong thuc `__call__`.

**Cach hoat dong:**
- Key = sha256(model + messages + kwargs) — `lm.py:407-410`
- Truoc khi goi LLM, check cache de lay cached responses — `lm.py:138-149`
- Responses uncached duoc xu ly va luu vao cache — `lm.py:168-171`

**Cac implementation:**
- `InMemoryCache`: OrderedDict LRU eviction — `cache.py:247-268`
- `SQLiteCache`: SQLite-backed persistent cache — `cache.py:168-244`
- Default: `InMemoryCache(max_size=1024)` — `cache.py:146-147`

---

## 4. Batching (litellm batch_completion)

**LM batching:**
- Default `max_batch_size=64` — `lm.py:73`
- Su dung `litellm.batch_completion` voi `max_workers=max_batch_size` — `lm.py:250-252`
- Rate-limited batching: chia sub-batches + sleep — `lm.py:258-303`
- TPM-limited batching: uoc luong token budget moi batch — `lm.py:311-390`

**Embedding batching:**
- `SentenceTransformersRM`: batch voi `max_batch_size=64` — `sentence_transformers_rm.py:29`
- `LiteLLMRM`: batch voi `max_batch_size=64` — `litellm_rm.py:28`

---

## 5. Rate/TPM Limiting

**Rate limiting (RPM):**
- `rate_limit` parameter tren LM constructor — `lm.py:74`
- Delay = `60 / rate_limit` giua cac batch — `lm.py:106`
- `max_batch_size` duoc cap boi `rate_limit` — `lm.py:108`
- Implementation: `_process_with_rate_limiting` — `lm.py:258-303`

**TPM limiting:**
- `tpm_limit` parameter — `lm.py:75`
- 95% safety buffer — `lm.py:317`
- Token usage history tracking voi 60-second window — `lm.py:305-309`
- Per-request token estimation qua `count_tokens` — `lm.py:319-321`
- Cho khi token window giai phong neu het budget — `lm.py:380-388`

---

## 6. Join Optimization (Search-Filter vs Map-Search-Filter)

**Vi tri:** `join_optimizer` tai `sem_join.py:417-527`.

**Day la query planner DUY NHAT thuc su trong LOTUS.**

**Hai strategies:**
1. **Search-Filter:** sem_sim_join truc tiep — `sem_join.py:463`
2. **Map-Search-Filter:** map l1 truoc (sem_map), roi sem_sim_join — `sem_join.py:483-486`

**Cach chon:**
- Tinh cascade thresholds cho ca hai strategies
- Dem so LLM calls can thiet cho low-confidence items
- Chon plan co it LLM calls hon — `sem_join.py:518-527`

---

## 7. Embedding-Based Pre-filtering cho TopK

**Vi tri:** `sem_topk.py:782-788` (method `quick-sem`).

**Cach hoat dong:**
- Truoc khi sorting, dung `sem_index` + `sem_search` de pre-sort DataFrame theo embedding similarity
- Pivot selection dung `heapq.nsmallest` thay vi random — `sem_topk.py:411-417`
- Giam so LLM comparison calls vi data da duoc pre-sorted

---

## Bang tom tat

| Ky thuat | Ap dung cho | Muc tieu chinh | File chinh |
|----------|-------------|----------------|------------|
| Model Cascading | filter, join, topk | Giam LLM calls 50-90% | cascade_utils.py, sem_filter.py, sem_join.py, sem_topk.py |
| Operator Cache | Tat ca operators | Tranh tinh toan lap lai | cache.py:33 |
| LM Response Cache | Moi LLM call | Tranh duplicate API calls | lm.py:136, cache.py:115 |
| Batch Processing | LM, RM | Tang throughput | lm.py:250, sentence_transformers_rm.py:67 |
| Rate/TPM Limiting | LM | Tuong thich API limits | lm.py:258, lm.py:311 |
| Join Optimizer | sem_join | Chon strategy re nhat | sem_join.py:417 |
| Embedding Pre-filter | sem_topk | Giam LLM comparisons | sem_topk.py:782 |
