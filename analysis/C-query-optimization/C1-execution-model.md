# C1 - Mô hình Execution của LOTUS

## Tổng quan

LOTUS không có query planner toàn cục. Các operator thực thi **eager, trái-sang-phải** theo chuỗi method calls trên DataFrame.
Mọi operator độc lập gọi LM hoặc retrieval model, xử lý kết quả, và trả về DataFrame mới.

---

## 1. Eager Execution — Không có Query Plan

**Không có IR (Intermediate Representation):**
- Mọi operator là một pandas DataFrame accessor — đăng ký qua `@pd.api.extensions.register_dataframe_accessor`
- Ví dụ: `sem_filter.py:225`, `sem_map.py:121`, `sem_topk.py:624`, `sem_agg.py:226`, `sem_join.py:606`
- Khi user viết `df.sem_filter(...).sem_map(...)`, Python thực thi tuần tự: `sem_filter` chạy xong, trả về DataFrame mới, rồi `sem_map` chạy trên kết quả đó

**Hệ quả:**
- Không có cơ hội reorder operators (ví dụ: push filter xuống trước join)
- Không có cost-based optimization toàn cục
- Mọi operator phải hoạt động độc lập, không biết gì về operator tiếp theo

---

## 2. Luồng xử lý trong mỗi Operator

Mọi operator semantic thực hiện các bước tương tự:

### Bước 1: Parse langex và validate columns
```
col_li = lotus.nl_expression.parse_cols(user_instruction)
# nl_expression.py:4-8 — regex match {column_name}
```
Kiểm tra columns tồn tại trong DataFrame — ví dụ `sem_filter.py:363-365`.

### Bước 2: Chuyển DataFrame thành multimodal data
```
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
# Mọi row -> dict chứa text + images
```
Ví dụ tại `sem_filter.py:367`, `sem_map.py:241`, `sem_topk.py:790`.

### Bước 3: Format user instruction
```
formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)
# nl_expression.py:17-21 — thay {col} bằng "Col"
```

### Bước 4: Tạo prompts và gọi LM
- Mọi operator có formatter riêng (filter_formatter, map_formatter, extract_formatter)
- Gửi batch prompts đến `model(inputs)` — gọi `LM.__call__` tại `lm.py:123`
- LM xử lý batching nội bộ — `lm.py:250-252`

### Bước 5: Post-process và trả về DataFrame
- `filter_postprocess` tại `postprocessors.py:182-218`: parse True/False
- `map_postprocess` tại `postprocessors.py:123-146`: raw output hoặc CoT extraction
- `extract_postprocess` tại `postprocessors.py:149-179`: JSON parsing
- Kết quả được gán vào DataFrame mới và trả về

---

## 3. GroupBy Parallelism qua ThreadPoolExecutor

**Các operator hỗ trợ group_by:**
- `sem_agg`: `sem_agg.py:381-399` — grouped aggregation
- `sem_topk`: `sem_topk.py:763-780` — grouped top-k

**Cách hoạt động:**
```python
# sem_agg.py:396-399
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

- `process_group` là static method xử lý mỗi group độc lập — `sem_agg.py:325-351`
- Max threads cấu hình tại `settings.py:23`: `parallel_groupby_max_threads: int = 8`
- Tương tự cho `sem_topk.py:770-773`

**Lưu ý:** Đây là thread-level parallelism, không phải process-level. Do GIL của Python, hiệu quả chủ yếu đến từ I/O-bound work (API calls).

---

## 4. Batch Processing trong LM

**Trong `LM.__call__`** tại `lm.py:123-190`:

1. **Check cache** — tách cached vs uncached messages — `lm.py:136-158`
2. **Xử lý uncached** qua `_process_uncached_messages` — `lm.py:215-256`:
   - Không có rate/tpm limit: `batch_completion(model, batch, max_workers=max_batch_size)` — `lm.py:250-252`
   - Có rate limit: chia sub-batches, sleep giữa các batch — `lm.py:258-303`
   - Có tpm limit: tính token budget, gửi từng sub-batch vừa đủ — `lm.py:311-390`
3. **Merge responses** — `lm.py:412-417`
4. **Extract outputs** — `lm.py:185-190`

**max_batch_size** mặc định 64 — `lm.py:73`. Nếu có `rate_limit`, bị cấp bởi `min(rate_limit, max_batch_size)` — `lm.py:108`.

---

## 5. Hierarchical Aggregation Tree

`sem_agg` có execution model đặc biệt — không chỉ gọi LM một lần.

**Tại `sem_agg.py:60-223`:**
- Docs được nhóm lại cho đến khi vừa context window
- Mỗi nhóm tạo một prompt, gửi đến LM
- Kết quả (summaries) trở thành input cho level tiếp theo
- Lặp lại cho đến khi chỉ còn 1 summary — `sem_agg.py:164`
- Sử dụng template khác nhau cho leaf vs node — `sem_agg.py:12-57`

**Độ phức tạp:** O(n / context_capacity) LLM calls mỗi level, O(log n) levels => O(n * log(n) / context_capacity) tổng LLM calls.

---

## 6. Sơ đồ luồng xử lý tổng thể

```
User Code: df.sem_filter("...").sem_map("...").sem_topk("...", K=5)
            |                    |                    |
            v                    v                    v
     [sem_filter.__call__]  [sem_map.__call__]  [sem_topk.__call__]
            |                    |                    |
     1. Parse langex       1. Parse langex      1. Parse langex
     2. df -> multimodal   2. df -> multimodal  2. df -> multimodal
     3. Format prompts     3. Format prompts    3. Build pairs
     4. LM(batch)          4. LM(batch)         4. LM(comparisons)
        |                     |                    |
        v                     v                    v
     [LM.__call__]         [LM.__call__]        [LM.__call__]
     - Check cache         - Check cache        - Check cache
     - batch_completion    - batch_completion   - batch_completion
     - Update stats        - Update stats       - Update stats
        |                     |                    |
        v                     v                    v
     5. Postprocess        5. Postprocess       5. Sort indexes
     6. Return DataFrame   6. Return DataFrame  6. Return DataFrame
```

**Key insight:** Mọi operator là một "island" độc lập. Không có data flow optimization giữa các operators.
