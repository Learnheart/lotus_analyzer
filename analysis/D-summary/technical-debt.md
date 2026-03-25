# Technical Debt — LOTUS

## Cac van de tim thay trong codebase

---

## 1. Settings khong Thread-Safe

**Vi tri:** `settings.py:5`

```python
# NOTE: Settings class is not thread-safe
```

**Van de:**
- `Settings` la class-level attributes (khong phai instance) — `settings.py:8-23`
- `settings = Settings()` la singleton global — `settings.py:35`
- Khi dung `ThreadPoolExecutor` cho group_by (vi du `sem_agg.py:396-399`, `sem_topk.py:770-773`), nhieu threads doc/ghi cung settings object
- Neu mot thread thay doi `lotus.settings.lm` trong khi thread khac dang su dung no → race condition
- `configure` method khong co lock — `settings.py:25-29`

**Muc do:** TRUNG BINH — hien tai cac threads chi doc settings, chua co case ghi dong thoi.

---

## 2. sem_dedup Tao Redundant Self-Join

**Vi tri:** `sem_dedup.py:45`

```python
joined_df = self._obj.sem_sim_join(self._obj, col_name, col_name, len(self._obj), lsuffix="_l", rsuffix="_r")
```

**Van de:**
- Join DataFrame voi chinh no: O(n^2) pairs
- Moi item duoc so sanh voi tat ca items khac, bao gom chinh no
- Phai filter ra self-matches sau — `sem_dedup.py:47`
- Voi n=1000 items, tao 1,000,000 pairs — rat cham va ton bo nho
- Cach tot hon: dung clustering hoac LSH de tim candidates truoc

**Muc do:** CAO — khong scale duoc voi dataset lon.

---

## 3. Khong co Streaming Support cho Large Datasets

**Van de:**
- Tat ca operators load toan bo data vao memory truoc khi xu ly
- `df2multimodal_info` chuyen TOAN BO DataFrame thanh list of dicts — vi du `sem_filter.py:367`
- Voi DataFrame co 100K rows va nhieu images, bo nho se bi day
- Khong co lazy evaluation hay pagination
- Khong co co che xu ly tung batch cua input data

**Muc do:** TRUNG BINH — phu thuoc vao kich thuoc dataset.

---

## 4. ColBERTv2RM get_vectors_from_index Khong Implement

**Vi tri:** `colbertv2_rm.py:95-109`

```python
def get_vectors_from_index(self, index_dir, ids):
    raise NotImplementedError("This method is not implemented for ColBERTv2RM")
```

**Van de:**
- `sem_sim_join.py:114-117` co try/except de handle:
  ```python
  try:
      queries = vs.get_vectors_from_index(query_index_dir, self._obj.index)
  except NotImplementedError:
      queries = self._obj[left_on]
  ```
- Khi fallback, queries la text thay vi pre-computed vectors → cham hon vi phai re-embed
- ColBERTv2 dung multi-vector representation nen khong the export single vector per doc

**Muc do:** THAP — co fallback hoat dong, chi cham hon.

---

## 5. Khong co Incremental Indexing

**Van de:**
- `sem_index` tao index moi tu dau moi lan — khong co `add_documents` method
- Khi them data moi vao DataFrame, phai re-index toan bo
- Vector stores (Faiss, Qdrant, Weaviate) ho tro incremental add, nhung LOTUS khong expose
- Khong co co che update/delete individual vectors

**Muc do:** TRUNG BINH — anh huong den workflow iterative.

---

## 6. safe_mode Khong Day Du

**Van de:**
Mot so operators chua implement safe_mode:

- `sem_agg.py:152-153`:
  ```python
  if safe_mode:
      lotus.logger.warning("Safe mode is not implemented yet")
  ```

- `sem_join.py:262-264` (cascade mode):
  ```python
  if safe_mode:
      lotus.logger.warning("Safe mode is not implemented yet.")
  ```

- `sem_dedup`, `sem_cluster_by`, `sem_partition_by`: khong co parameter safe_mode

**Muc do:** THAP — safe_mode la optional feature.

---

## 7. Tat ca Extract Values Bi Cast thanh str

**Vi tri:** `postprocessors.py:176-177`

```python
output = {key: str(value) for key, value in output.items()}
```

Tuong tu tai `postprocessors.py:38`:
```python
json_obj = {key: str(value) for key, value in json_obj.items()}
```

Va tai `postprocessors.py:88`:
```python
json_obj = {key: str(value) for key, value in json_obj.items()}
```

**Van de:**
- LLM tra ve JSON voi numbers, booleans, lists — tat ca deu bi convert thanh string
- `{"rating": 5, "is_positive": true}` tro thanh `{"rating": "5", "is_positive": "True"}`
- User phai tu cast lai types sau khi extract
- Mat thong tin type information tu LLM output

**Muc do:** TRUNG BINH — de fix nhung anh huong den usability.

---

## 8. InMemoryCache Khong Phai LRU Thuc Su

**Vi tri:** `cache.py:247-268`

```python
def get(self, key):
    return self.cache.get(key)  # Khong move_to_end!

def insert(self, key, value):
    self.cache[key] = value
    if len(self.cache) > self.max_size:
        self.cache.popitem(last=False)  # Evict oldest by insertion order
```

**Van de:**
- `get()` tai `cache.py:252-256` khong goi `self.cache.move_to_end(key)`
- Items duoc evict theo **insertion order**, khong phai **access order**
- Day la FIFO, khong phai LRU
- Items duoc access nhieu van co the bi evict som

**Muc do:** THAP — cache van hoat dong, chi khong optimal.

---

## 9. HeapDoc Dung Class Variables cho State

**Vi tri:** `sem_topk.py:507-511`

```python
class HeapDoc:
    num_calls: int = 0
    total_tokens: int = 0
    strategy: ReasoningStrategy | None = None
    model: lotus.models.LM | None = None
    explanations: dict[int, list[str]] = {}
```

**Van de:**
- Class variables duoc chia se giua TAT CA instances
- Phai reset manual truoc moi lan dung — `sem_topk.py:605-609`
- Khong thread-safe — neu hai sem_topk chay song song (group_by), se ghi de len nhau
- Mutable default `explanations = {}` duoc chia se giua tat ca invocations

**Muc do:** TRUNG BINH — co the gay loi kho debug trong parallel execution.

---

## 10. Error Handling trong Cascade Fallback

**Vi tri:** `sem_join.py:598-601`

```python
except Exception as e:
    lotus.logger.error(f"Error while learning filter cascade thresholds: {e}")
    lotus.logger.error("Default to full join.")
    return 1.0, 0.0, len(sample_indices)
```

**Van de:**
- Catch `Exception` qua rong — bat ca programming errors
- Fallback ve full join (thresholds 1.0, 0.0) co the rat dat
- User khong duoc canh bao ve chi phi tang dot ngot
- Tuong tu tai `sem_filter.py:220-222` — raise lai exception nhung da mat context

**Muc do:** THAP — failsafe behavior hop ly, chi can log tot hon.

---

## Tom tat theo Muc do

| Muc do | Van de | File chinh |
|--------|--------|-----------|
| CAO | sem_dedup self-join O(n^2) | sem_dedup.py:45 |
| TRUNG BINH | Settings khong thread-safe | settings.py:5 |
| TRUNG BINH | Khong co streaming | Toan bo codebase |
| TRUNG BINH | Khong co incremental indexing | sem_index.py |
| TRUNG BINH | Extract values cast str | postprocessors.py:176 |
| TRUNG BINH | HeapDoc class variables | sem_topk.py:507-511 |
| THAP | ColBERTv2 get_vectors | colbertv2_rm.py:109 |
| THAP | safe_mode khong day du | sem_agg.py:152, sem_join.py:262 |
| THAP | InMemoryCache khong LRU thuc su | cache.py:252 |
| THAP | Cascade error handling rong | sem_join.py:598 |
