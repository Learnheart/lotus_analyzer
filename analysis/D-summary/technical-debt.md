# Technical Debt — LOTUS

## Các vấn đề tìm thấy trong codebase

---

## 1. Settings không Thread-Safe

**Vị trí:** `settings.py:5`

```python
# NOTE: Settings class is not thread-safe
```

**Vấn đề:**
- `Settings` là class-level attributes (không phải instance) — `settings.py:8-23`
- `settings = Settings()` là singleton global — `settings.py:35`
- Khi dùng `ThreadPoolExecutor` cho group_by (ví dụ `sem_agg.py:396-399`, `sem_topk.py:770-773`), nhiều threads đọc/ghi cùng settings object
- Nếu một thread thay đổi `lotus.settings.lm` trong khi thread khác đang sử dụng nó → race condition
- `configure` method không có lock — `settings.py:25-29`

**Mức độ:** TRUNG BÌNH — hiện tại các threads chỉ đọc settings, chưa có case ghi đồng thời.

---

## 2. sem_dedup Tạo Redundant Self-Join

**Vị trí:** `sem_dedup.py:45`

```python
joined_df = self._obj.sem_sim_join(self._obj, col_name, col_name, len(self._obj), lsuffix="_l", rsuffix="_r")
```

**Vấn đề:**
- Join DataFrame với chính nó: O(n^2) pairs
- Mỗi item được so sánh với tất cả items khác, bao gồm chính nó
- Phải filter ra self-matches sau — `sem_dedup.py:47`
- Với n=1000 items, tạo 1,000,000 pairs — rất chậm và tốn bộ nhớ
- Cách tốt hơn: dùng clustering hoặc LSH để tìm candidates trước

**Mức độ:** CAO — không scale được với dataset lớn.

---

## 3. Không có Streaming Support cho Large Datasets

**Vấn đề:**
- Tất cả operators load toàn bộ data vào memory trước khi xử lý
- `df2multimodal_info` chuyển TOÀN BỘ DataFrame thành list of dicts — ví dụ `sem_filter.py:367`
- Với DataFrame có 100K rows và nhiều images, bộ nhớ sẽ bị đầy
- Không có lazy evaluation hay pagination
- Không có cơ chế xử lý từng batch của input data

**Mức độ:** TRUNG BÌNH — phụ thuộc vào kích thước dataset.

---

## 4. ColBERTv2RM get_vectors_from_index Không Implement

**Vị trí:** `colbertv2_rm.py:95-109`

```python
def get_vectors_from_index(self, index_dir, ids):
    raise NotImplementedError("This method is not implemented for ColBERTv2RM")
```

**Vấn đề:**
- `sem_sim_join.py:114-117` có try/except để handle:
  ```python
  try:
      queries = vs.get_vectors_from_index(query_index_dir, self._obj.index)
  except NotImplementedError:
      queries = self._obj[left_on]
  ```
- Khi fallback, queries là text thay vì pre-computed vectors → chậm hơn vì phải re-embed
- ColBERTv2 dùng multi-vector representation nên không thể export single vector per doc

**Mức độ:** THẤP — có fallback hoạt động, chỉ chậm hơn.

---

## 5. Không có Incremental Indexing

**Vấn đề:**
- `sem_index` tạo index mới từ đầu mỗi lần — không có `add_documents` method
- Khi thêm data mới vào DataFrame, phải re-index toàn bộ
- Vector stores (Faiss, Qdrant, Weaviate) hỗ trợ incremental add, nhưng LOTUS không expose
- Không có cơ chế update/delete individual vectors

**Mức độ:** TRUNG BÌNH — ảnh hưởng đến workflow iterative.

---

## 6. safe_mode Không Đầy Đủ

**Vấn đề:**
Một số operators chưa implement safe_mode:

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

- `sem_dedup`, `sem_cluster_by`, `sem_partition_by`: không có parameter safe_mode

**Mức độ:** THẤP — safe_mode là optional feature.

---

## 7. Tất cả Extract Values Bị Cast thành str

**Vị trí:** `postprocessors.py:176-177`

```python
output = {key: str(value) for key, value in output.items()}
```

Tương tự tại `postprocessors.py:38`:
```python
json_obj = {key: str(value) for key, value in json_obj.items()}
```

Và tại `postprocessors.py:88`:
```python
json_obj = {key: str(value) for key, value in json_obj.items()}
```

**Vấn đề:**
- LLM trả về JSON với numbers, booleans, lists — tất cả đều bị convert thành string
- `{"rating": 5, "is_positive": true}` trở thành `{"rating": "5", "is_positive": "True"}`
- User phải tự cast lại types sau khi extract
- Mất thông tin type information từ LLM output

**Mức độ:** TRUNG BÌNH — dễ fix nhưng ảnh hưởng đến usability.

---

## 8. InMemoryCache Không Phải LRU Thực Sự

**Vị trí:** `cache.py:247-268`

```python
def get(self, key):
    return self.cache.get(key)  # Không move_to_end!

def insert(self, key, value):
    self.cache[key] = value
    if len(self.cache) > self.max_size:
        self.cache.popitem(last=False)  # Evict oldest by insertion order
```

**Vấn đề:**
- `get()` tại `cache.py:252-256` không gọi `self.cache.move_to_end(key)`
- Items được evict theo **insertion order**, không phải **access order**
- Đây là FIFO, không phải LRU
- Items được access nhiều vẫn có thể bị evict sớm

**Mức độ:** THẤP — cache vẫn hoạt động, chỉ không optimal.

---

## 9. HeapDoc Dùng Class Variables cho State

**Vị trí:** `sem_topk.py:507-511`

```python
class HeapDoc:
    num_calls: int = 0
    total_tokens: int = 0
    strategy: ReasoningStrategy | None = None
    model: lotus.models.LM | None = None
    explanations: dict[int, list[str]] = {}
```

**Vấn đề:**
- Class variables được chia sẻ giữa TẤT CẢ instances
- Phải reset manual trước mỗi lần dùng — `sem_topk.py:605-609`
- Không thread-safe — nếu hai sem_topk chạy song song (group_by), sẽ ghi đè lên nhau
- Mutable default `explanations = {}` được chia sẻ giữa tất cả invocations

**Mức độ:** TRUNG BÌNH — có thể gây lỗi khó debug trong parallel execution.

---

## 10. Error Handling trong Cascade Fallback

**Vị trí:** `sem_join.py:598-601`

```python
except Exception as e:
    lotus.logger.error(f"Error while learning filter cascade thresholds: {e}")
    lotus.logger.error("Default to full join.")
    return 1.0, 0.0, len(sample_indices)
```

**Vấn đề:**
- Catch `Exception` quá rộng — bắt cả programming errors
- Fallback về full join (thresholds 1.0, 0.0) có thể rất đắt
- User không được cảnh báo về chi phí tăng đột ngột
- Tương tự tại `sem_filter.py:220-222` — raise lại exception nhưng đã mất context

**Mức độ:** THẤP — failsafe behavior hợp lý, chỉ cần log tốt hơn.

---

## Tóm tắt theo Mức độ

| Mức độ | Vấn đề | File chính |
|--------|--------|-----------|
| CAO | sem_dedup self-join O(n^2) | sem_dedup.py:45 |
| TRUNG BÌNH | Settings không thread-safe | settings.py:5 |
| TRUNG BÌNH | Không có streaming | Toàn bộ codebase |
| TRUNG BÌNH | Không có incremental indexing | sem_index.py |
| TRUNG BÌNH | Extract values cast str | postprocessors.py:176 |
| TRUNG BÌNH | HeapDoc class variables | sem_topk.py:507-511 |
| THẤP | ColBERTv2 get_vectors | colbertv2_rm.py:109 |
| THẤP | safe_mode không đầy đủ | sem_agg.py:152, sem_join.py:262 |
| THẤP | InMemoryCache không LRU thực sự | cache.py:252 |
| THẤP | Cascade error handling rộng | sem_join.py:598 |
