# C2 - Query Planner trong LOTUS (Hạn chế)

## Tổng quan

LOTUS có **query planning RẤT HẠN CHẾ**. Chỉ có duy nhất `join_optimizer` tại `sem_join.py:417` thực sự là một optimizer.
Không có global query plan nào cho chuỗi các operators được chain với nhau.

---

## 1. join_optimizer — Optimizer duy nhất

### Vị trí
`sem_join.py:417-527`

### Chức năng
So sánh hai strategies để tìm plan rẻ nhất, đảm bảo recall và precision target.

### Hai strategies được so sánh

#### Strategy 1: Search-Filter (SF)
1. Chạy `run_sem_sim_join(l1, l2)` — `sem_join.py:463`
   - Dùng embedding similarity làm proxy scores
   - `sem_sim_join` trả về DataFrame với `_scores` column
2. `calibrate_sem_sim_join` — `cascade_utils.py:147-149` — clip scores về [0, 1]
3. `learn_join_cascade_threshold` — `sem_join.py:530-603`
   - Sample một phần data (`importance_sampling` — `cascade_utils.py:8-30`)
   - Chạy oracle LM trên sample — `sem_join.py:577-587`
   - Tìm optimal (pos_threshold, neg_threshold) — `sem_join.py:589-594`
4. Phân loại:
   - `_scores >= pos_threshold` → accept (high confidence positive) — `sem_join.py:477`
   - `_scores <= neg_threshold` → reject (high confidence negative) — `sem_join.py:478`
   - Còn lại → cần oracle LM — `sem_join.py:479-480`

#### Strategy 2: Map-Search-Filter (MSF)
1. `map_l1_to_l2(l1, col1_label, col2_label)` — `sem_join.py:369-414`
   - Dùng `sem_map` để "dịch" l1 sang domain của l2
   - Ví dụ: "Cho product name, liệt kê 2-10 categories liên quan"
   - Chi phí: len(l1) LLM calls — `sem_join.py:504`
2. Chạy `run_sem_sim_join` trên mapped l1 — `sem_join.py:486`
3. Tương tự SF: learn thresholds, phân loại high/low confidence

### Quyết định chọn plan

```python
# sem_join.py:518-527
if sf_cost < msf_cost:
    lotus.logger.info("Proceeding with Search-Filter")
    return sf_high_conf, sf_low_conf, sf_high_conf_neg, learning_cost
else:
    lotus.logger.info("Proceeding with Map-Search-Filter")
    return msf_high_conf, msf_low_conf, msf_high_conf_neg, learning_cost
```

- `sf_cost = len(sf_low_conf)` — số LLM calls cho Search-Filter — `sem_join.py:480`
- `msf_cost = len(msf_low_conf)` — số LLM calls cho Map-Search-Filter — `sem_join.py:503`
- Không tính đến chi phí learning (đã bỏ ra rồi)
- Chi phí MSF có thêm `len(l1)` từ bước map — `sem_join.py:504`

### Log output
```python
# sem_join.py:507-515
lotus.logger.info("Join Optimizer: plan cost analysis:")
lotus.logger.info(f"    Search-Filter: {sf_cost} LLM calls.")
lotus.logger.info(f"    Map-Search-Filter: {msf_cost} LLM calls.")
```

---

## 2. Không có Global Query Plan

### Vấn đề
Khi user viết:
```python
df.sem_filter("Is {text} about AI?")
  .sem_map("Summarize {text}")
  .sem_topk("Most relevant {_map}", K=5)
```

**Mọi operator chạy độc lập, không biết về operator sau:**
- `sem_filter` gọi LLM cho **tất cả** rows trước
- `sem_map` gọi LLM cho tất cả rows còn lại
- `sem_topk` gọi LLM cho quicksort comparisons

**Nhưng một optimizer thông minh có thể:**
- Push `sem_topk` vào trước `sem_map` (vì sem_topk chỉ cần K rows)
- Kết hợp `sem_filter` + `sem_map` thành một LLM call
- Pre-filter bằng embedding trước khi chạy bất kỳ LLM operator nào

### Nguyên nhân
- LOTUS dùng pandas accessor pattern — mọi operator là một method call độc lập
- Không có IR (Intermediate Representation) hay execution graph
- Không có lazy evaluation — tất cả là eager execution
- Thiết kế ưu tiên đơn giản và dễ hiểu hơn là tối ưu hiệu suất

---

## 3. Cascade Threshold Learning — Planning nội bộ

Dù không có global planner, mỗi cascade operator có "micro-planning" nội bộ:

### Filter cascade planning — `sem_filter.py:395-462`
1. Chạy proxy model (helper_lm hoặc embedding) trên TOÀN BỘ data
2. `importance_sampling` để chọn sample representative — `cascade_utils.py:8-30`
3. Chạy oracle LM trên sample — `sem_filter.py:196-208`
4. `learn_cascade_thresholds` để tìm optimal thresholds — `cascade_utils.py:42-144`
5. Áp dụng thresholds: high-confidence → proxy, low-confidence → oracle — `sem_filter.py:467-530`

### Join cascade planning — `sem_join.py:242-257`
1. `join_optimizer` chạy CẢ HAI strategies và chọn rẻ hơn
2. Mỗi strategy có `learn_join_cascade_threshold` riêng — `sem_join.py:464-476` và `sem_join.py:487-499`

### TopK cascade — `sem_topk.py:401-455`
1. Không có threshold learning trước
2. Cascade được áp dụng on-the-fly trong mỗi partition step
3. Helper LM chạy trước, kiểm tra confidence trên logprobs — `sem_topk.py:248-252`
4. Low-confidence comparisons gửi đến large LM — `sem_topk.py:257-272`

---

## 4. safe_mode — Ước lượng chi phí (Không phải Optimizer)

`safe_mode` cho phép ước lượng chi phí trước khi thực thi, nhưng **không thay đổi execution plan**.

**Hỗ trợ:**
- `sem_filter`: `sem_filter.py:107-110` — ước lượng token count cho tất cả inputs
- `sem_join`: `sem_join.py:104-120` — ước lượng len(l1) * len(l2) calls
- `sem_topk` quicksort: `sem_topk.py:393-399` — ước lượng 2K + 2N*log(N) calls
- `sem_topk` heapsort: `sem_topk.py:597-603` — ước lượng N*log(N) + K*log(N) calls

**KHÔNG hỗ trợ (TODO):**
- `sem_agg`: `sem_agg.py:152-153` — "Safe mode is not implemented yet"
- `sem_join` cascade: `sem_join.py:262-264` — "Safe mode is not implemented yet"

---

## 5. Tóm tắt

| Khía cạnh | Trạng thái | Chi tiết |
|-----------|-----------|----------|
| Global query planner | KHÔNG CÓ | Eager execution, trái-sang-phải |
| Join optimizer | CÓ | `sem_join.py:417` — SF vs MSF |
| Filter cascade | CÓ | Threshold learning + routing |
| TopK cascade | CÓ | On-the-fly confidence routing |
| Operator reordering | KHÔNG CÓ | Không có IR/graph |
| Cost estimation | MỘT PHẦN | safe_mode cho một số operators |
| Predicate pushdown | KHÔNG CÓ | Không có cơ chế nào |
| Common subexpression elimination | KHÔNG CÓ | Tuy operator_cache có thể cover một phần |
