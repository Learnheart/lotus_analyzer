# E0 - Data Philosophy: Semantic Operator Model

## Tổng quan triết lý

LOTUS (LLMs Over Tables of Unstructured and Structured data) xây dựng một **Semantic Operator Model** - ánh xạ các toán tử quan hệ truyền thống sang các toán tử ngữ nghĩa sử dụng LLM. Triết lý cốt lõi: user khai báo ý định bằng ngôn ngữ tự nhiên (declarative), engine quyết định cách thực thi (imperative).

**Authors**: Liana Patel (Stanford, lianapat@stanford.edu), Siddharth Jha (Berkeley, sidjha@berkeley.edu) - `pyproject.toml:11-13`.

---

## 1. Semantic Operator Mapping

Mỗi toán tử SQL truyền thống có một phiên bản semantic tương ứng:

| SQL Operator | Semantic Operator | Cơ chế |
|---|---|---|
| `WHERE` | `sem_filter` | LLM đánh giá "claim" True/False cho mỗi row (`task_instructions.py:99-101`) |
| `JOIN` | `sem_join` | LLM đánh giá từng cặp (l1[i], l2[j]) qua sem_filter (`sem_join.py:137-147`) |
| `GROUP BY + AGG` | `sem_agg` | Hierarchical tree aggregation: leaf → node → summary (`sem_agg.py:60-223`) |
| `ORDER BY + LIMIT` | `sem_topk` | LLM pairwise comparison: "Document 1 or Document 2?" (`sem_topk.py:51-66`) |
| `SELECT (transform)` | `sem_map` | LLM trả lời instruction cho mỗi row |
| `SELECT (extract)` | `sem_extract` | LLM extract JSON fields từ context (`sem_extract.py:15-108`) |
| `LIKE / CONTAINS` | `sem_search` | Vector similarity search qua embedding (`sem_search.py:92-157`) |

### Điểm khác biệt quan trọng

Toán tử SQL truyền thống dựa trên **exact matching** (giá trị phải khớp chính xác). Semantic operators dựa trên **LLM reasoning** - cho phép fuzzy, contextual matching. Ví dụ:
- SQL: `WHERE category = 'AI'` → chỉ khớp string "AI"
- LOTUS: `sem_filter("the {article} belongs to AI")` → LLM suy luận "Machine learning tutorial" cũng thuộc AI

---

## 2. Declarative vs Imperative

### User declares "what" (Declarative)

User chỉ cần viết **langex** (language expression) với cú pháp `{column_name}`:

```python
df.sem_filter("the {description} mentions a positive sentiment")
df.sem_join(df2, "the {article} belongs to the {category}")
```

Langex được parse bởi `nl_expression.py:4-14` (hàm `parse_cols`) sử dụng regex `(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})` để trích xuất tên cột.

### Engine decides "how" (Imperative)

Engine tự động quyết định:

1. **Model selection**: Cascade routing giữa helper_lm (nhỏ, rẻ) và lm (lớn, đắt) dựa trên confidence threshold (`sem_filter.py:395-462`).

2. **Prompt engineering**: Tự động format context qua `df2text` (`task_instructions.py:325-361`) và `df2multimodal_info` (`task_instructions.py:364-379`), chọn template phù hợp (filter_formatter, map_formatter, extract_formatter).

3. **Join optimization**: `join_optimizer` (`sem_join.py:417-527`) tự động so sánh cost giữa hai join plan:
   - **Search-Filter**: sem_sim_join → threshold → LLM verify
   - **Map-Search-Filter**: sem_map l1 → sem_sim_join → threshold → LLM verify
   - Chọn plan có ít LLM calls hơn (`sem_join.py:518-527`)

4. **Threshold learning**: `learn_cascade_thresholds` (`cascade_utils.py:42-144`) tự động học threshold tối ưu từ sample data với statistical correction (UB/LB bounds) để đảm bảo recall/precision targets.

---

## 3. Evidence: Automatic Optimization

### Cascade Threshold Learning
```
cascade_utils.py:42-144 - learn_cascade_thresholds()
```
- Input: proxy_scores, oracle_outputs, correction_factors, cascade_args
- Output: (pos_threshold, neg_threshold) tối ưu
- Statistical correction: UB/LB bounds với failure_probability (`cascade_utils.py:52-56`)
- Recall target → tìm tau_neg, Precision target → tìm tau_pos (`cascade_utils.py:103-137`)

### Join Plan Auto-Selection
```
sem_join.py:417-527 - join_optimizer()
```
- Chạy cả hai plan trên sample data
- So sánh `sf_cost` vs `msf_cost` (số LLM calls cho low-confidence pairs)
- Log chi tiết: `sem_join.py:507-515`
- Tự động chọn plan rẻ hơn: `sem_join.py:518-527`

### Importance Sampling
```
cascade_utils.py:8-30 - importance_sampling()
```
- Weighted sampling: `w = sqrt(proxy_scores)` (`cascade_utils.py:16`)
- IS weight mixing: `is_weight * w/sum(w) + (1 - is_weight) * uniform` (`cascade_utils.py:18`)
- Sample size: `sampling_percentage * len(proxy_scores)` (`cascade_utils.py:25`)

---

## 4. Tóm tắt triết lý

LOTUS thể hiện một paradigm shift: từ "query data bằng exact matching" sang "query data bằng semantic reasoning". User chỉ cần mô tả ý định, engine tối ưu hóa execution plan. Đây là bước tiến tự nhiên khi dữ liệu ngày càng unstructured và cần reasoning thay vì chỉ pattern matching.
