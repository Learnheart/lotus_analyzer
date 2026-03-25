# Innovation Highlights — LOTUS

## Top 7 Innovations với File:Line References

---

## 1. Pandas Accessor Pattern cho Seamless DataFrame Integration

**Vị trí:** Tất cả operator files, ví dụ `sem_filter.py:225`, `sem_map.py:121`, `sem_topk.py:624`

**Innovation:**
LOTUS đăng ký mỗi semantic operator như một pandas DataFrame accessor qua `@pd.api.extensions.register_dataframe_accessor`. Điều này cho phép:

```python
# Tích hợp tự nhiên vào pandas workflow
df.sem_filter("Is {text} positive?").sem_map("Summarize {text}").sem_topk("Best {_map}", K=5)
```

**Tại sao đột phá:**
- Không cần DSL riêng hay API mới — dùng trực tiếp trên pandas
- Chainable — kết quả của operator này là input của operator tiếp theo
- Zero migration cost — DataFrame hiện tại vẫn dùng được
- Mỗi operator là một class với `__init__` nhận `pandas_obj` và `__call__` thực thi logic — ví dụ `sem_filter.py:308-316`

**Langex (Natural Language Expressions):**
User tham chiếu columns bằng `{column_name}` — parse bởi regex tại `nl_expression.py:4-8`:
```python
pattern = r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"
```

---

## 2. Model Cascading với Statistical Guarantees

**Vị trí:** `cascade_utils.py:42-144`, `sem_filter.py:383-530`, `sem_join.py:180-333`, `sem_topk.py:176-273`

**Innovation:**
Hệ thống tự động học cascade thresholds từ data, đảm bảo recall và precision targets với **đảm bảo thống kê (statistical guarantees)**.

**Cơ sở toán học:**
- Hoeffding-style concentration inequalities — `cascade_utils.py:52-56`
- Importance sampling với correction factors — `cascade_utils.py:8-30`
- Corrected recall target qua UB/LB bounds — `cascade_utils.py:118-121`
- Precision guarantee với Bonferroni correction — `cascade_utils.py:127-137`

**Tại sao đột phá:**
- Không chỉ là "dùng model nhỏ trước" — có lý thuyết đảm bảo độ chính xác
- User chỉ cần set `recall_target=0.9, precision_target=0.9` và hệ thống tự tìm thresholds
- Giảm chi phí 50-90% so với chạy full oracle model

---

## 3. Join Optimizer (Search-Filter vs Map-Search-Filter)

**Vị trí:** `sem_join.py:417-527`

**Innovation:**
LOTUS có query optimizer cho semantic join — so sánh hai strategies và chọn plan rẻ nhất:

1. **Search-Filter (SF):** Embedding similarity → filter uncertain pairs với oracle LLM
2. **Map-Search-Filter (MSF):** LLM map left table → embedding similarity → filter

**Quyết định:** `sem_join.py:518-527`
```python
if sf_cost < msf_cost:
    return sf_high_conf, sf_low_conf, sf_high_conf_neg, learning_cost
else:
    return msf_high_conf, msf_low_conf, msf_high_conf_neg, learning_cost
```

**Tại sao đột phá:**
- Đây là cost-based optimizer cho LLM queries — rất ít framework làm được
- Tự động chọn strategy tốt nhất dựa trên data characteristics
- Map step (MSF) có thể giảm search space đáng kể khi domains khác nhau nhiều

---

## 4. Hierarchical Aggregation Tree cho Unbounded Data

**Vị trí:** `sem_agg.py:60-223`

**Innovation:**
Khi data lớn hơn context window, sem_agg xây dựng cây aggregation đa cấp:
- Level 0: Nhóm documents thành batches vừa context window, mỗi batch → summary
- Level 1+: Nhóm summaries thành batches, mỗi batch → higher-level summary
- Lặp lại cho đến khi chỉ còn 1 final summary — `sem_agg.py:164`

**Template khác nhau cho mỗi level:**
- Leaf template — `sem_agg.py:12-31`: xử lý raw documents
- Node template — `sem_agg.py:34-57`: xử lý summaries từ các sources khác nhau

**Partition-aware:** Documents cùng partition_id được aggregation với nhau — `sem_agg.py:179-184`.

**Tại sao đột phá:**
- Xử lý dữ liệu KHÔNG GIỚI HẠN — không bị giới hạn bởi context window
- Đảm bảo thông tin từ tất cả documents đều được xem xét
- Hiệu quả: O(n * log(n) / context_capacity) LLM calls

---

## 5. Multimodal Support qua ImageDtype Extension

**Vị trí:** `dtype_extensions/image.py:12-35` (ImageDtype), `dtype_extensions/image.py:37-305` (ImageArray)

**Innovation:**
LOTUS tạo custom pandas ExtensionDtype cho images, cho phép lưu trữ và xử lý images trực tiếp trong DataFrame.

```python
class ImageDtype(ExtensionDtype):
    name = "image"
    type = Image.Image
    na_value = None
```

**Đặc điểm:**
- `ImageArray` hỗ trợ lazy loading và caching — `image.py:117-132`
- Comparison logic cho images — `image.py:307-327`
- Tự động convert giữa Image, base64, và file paths
- Tích hợp với multimodal LLM prompts qua `task_instructions.df2multimodal_info`

**Tại sao đột phá:**
- Không chỉ xử lý text — DataFrame có thể chứa cả images
- Images được xử lý như "first-class citizens" trong mỗi operator
- Một API thống nhất cho cả text và vision tasks

---

## 6. Langex — Natural Language Expressions làm Declarative Query Interface

**Vị trí:** `nl_expression.py:1-29`

**Innovation:**
Langex là cách LOTUS cho phép user viết queries bằng ngôn ngữ tự nhiên với column references:

```python
# Langex syntax
df.sem_filter("The {review} expresses a positive sentiment about {product}")
df.sem_map("Translate {text} to French")
df.sem_topk("The {title} is most relevant to machine learning", K=5)
```

**Cách hoạt động:**
1. `parse_cols` — `nl_expression.py:4-8`: extract column names từ `{...}`
2. `nle2str` — `nl_expression.py:17-21`: thay `{col}` bằng `Col.capitalize()`
3. Double braces `{{...}}` được ignore (escaped) — `nl_expression.py:6`

**Tại sao đột phá:**
- Declarative — user nói "cái gì" chứ không phải "làm thế nào"
- Natural language làm bridge giữa structured data (columns) và unstructured reasoning (LLM)
- Dễ học, dễ sử dụng — không cần học DSL phức tạp

---

## 7. LLM-Based Pairwise Comparison Algorithms (Quicksort, Heapsort)

**Vị trí:** `sem_topk.py:347-488` (quicksort), `sem_topk.py:560-621` (heapsort), `sem_topk.py:276-344` (naive sort)

**Innovation:**
LOTUS implement các thuật toán sorting cổ điển nhưng thay comparator function bằng LLM call:

**Quicksort** — `sem_topk.py:347-488`:
- Partition dùng LLM pairwise comparisons — `sem_topk.py:407-463`
- Chỉ sort top-K (không cần sort toàn bộ) — `sem_topk.py:465-483`
- Hỗ trợ cascade threshold — `sem_topk.py:437-455`
- Hỗ trợ embedding-based pivot selection — `sem_topk.py:411-417`

**Heapsort** — `sem_topk.py:560-621`:
- `HeapDoc` class override `__lt__` để gọi LLM — `sem_topk.py:526-557`
- `heapq.nsmallest(K, heap)` — chỉ lấy K items — `sem_topk.py:613`
- O(n + K log n) LLM calls

**Naive sort** — `sem_topk.py:276-344`:
- All-pairs comparison + voting — `sem_topk.py:310-339`
- O(n^2) LLM calls — chỉ dùng cho dataset nhỏ

**Tại sao đột phá:**
- Áp dụng computer science algorithms lên LLM reasoning
- Quicksort với top-K pruning giảm từ O(n^2) xuống O(n log n) avg
- Cascade + embedding pivot tiếp tục giảm chi phí
- Kết quả là ranking dựa trên LLM judgment — không phải embedding similarity
