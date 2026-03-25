# SEM_JOIN — `sem_join`

## Metadata
- **File**: `lotus/sem_ops/sem_join.py`
- **Accessor line**: 606 (`@pd.api.extensions.register_dataframe_accessor("sem_join")`)
- **Core function line**: 16 (`def sem_join(...)`)
- **Cascade function line**: 180 (`def sem_join_cascade(...)`)
- **Join optimizer line**: 417 (`def join_optimizer(...)`)
- **Type**: Binary operator
- **Requires**: `lotus.settings.lm` (sem_join.py:685-689)

## 1. Purpose & Use Cases

Join 2 DataFrames dựa trên điều kiện ngữ nghĩa. Mỗi cặp (left_row, right_row) được đánh giá True/False bởi LLM. Độ phức tạp M x N.

**Use cases:**
- Category matching: `df1.sem_join(df2, "the {article} belongs to the {category}")`
- Entity resolution: `df1.sem_join(df2, "the {product:left} is the same as {item:right}")`
- Relationship discovery: `df1.sem_join(df2, "the {person} is mentioned in {document}")`

## 2. Call Stack Trace

```
1. SemJoinDataframe.__call__()                          # sem_join.py:670
2.   lotus.nl_expression.parse_cols(join_instruction)   # sem_join.py:699
3.   Xác định left_on và right_on columns               # sem_join.py:700-730
4.   [Nếu có examples]: df2multimodal_info(examples)    # sem_join.py:737
5.   [Nếu cascade_args != None và đủ lớn]:
5a.    sem_join_cascade()                                # sem_join.py:755-773
5b.      join_optimizer()                                # sem_join.py:242-257
5c.        run_sem_sim_join() cho SF plan                # sem_join.py:463
5d.        learn_join_cascade_threshold() cho SF         # sem_join.py:464-476
5e.        map_l1_to_l2() cho MSF plan                  # sem_join.py:483-485
5f.        run_sem_sim_join() cho MSF plan               # sem_join.py:486
5g.        learn_join_cascade_threshold() cho MSF        # sem_join.py:487-498
5h.        Chọn plan rẻ hơn                              # sem_join.py:518-527
5i.      Accept high confidence results                  # sem_join.py:267
5j.      sem_filter() cho low confidence                 # sem_join.py:291-301
6.   [Nếu không cascade]:
6a.    sem_join()                                        # sem_join.py:775-791
6b.      Core sem_join():
6c.        df2multimodal_info() cho left và right        # sem_join.py:101-102
6d.        merge_multimodal_info() tạo pairs             # sem_join.py:132
6e.        sem_filter() trên tất cả pairs                # sem_join.py:137-147
7.   Build joined DataFrame                             # sem_join.py:798-817
8.   Return DataFrame                                   # sem_join.py:822
```

## 3. Prompt Template (COPY VERBATIM)

sem_join sử dụng **filter_formatter** (task_instructions.py:87-157) nội bộ. Prompt giống hệt sem_filter:

### System instruction (task_instructions.py:99-101):
```
The user will provide a claim and some relevant context.
    Your job is to determine whether the claim is true for the given context.
```

### User message format:
```
Context:
[Left_col]: «left_value»
[Right_col]: «right_value»

Claim: {join_instruction}
```

## 4. LLM Interaction

- **Core mechanism**: sem_join gọi `sem_filter()` nội bộ (sem_join.py:137-147)
- **M x N complexity**: Mỗi cặp (left, right) tạo 1 doc rồi gửi qua sem_filter (sem_join.py:128-135)
- **merge_multimodal_info**: Gộp left + right multimodal data (task_instructions.py:382-402)
- **Cascade**: join_optimizer so sánh 2 plans (SF vs MSF) và chọn plan rẻ hơn (sem_join.py:506-527)

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | Tất cả M*N pairs gửi 1 lần qua sem_filter (sem_join.py:137) |
| Caching | Yes | `@operator_cache` (sem_join.py:669) |
| Cascading | Yes | join_optimizer so sánh SF vs MSF plans (sem_join.py:417-527) |
| Early-termination | No | |
| Sampling | Yes | importance_sampling trong learn_join_cascade_threshold (sem_join.py:566) |
| Safe mode | Partial | Chỉ estimate cho non-cascade path (sem_join.py:104-120) |

### Join Optimizer chi tiết (sem_join.py:417-527):

**Search-Filter (SF) plan:**
1. `run_sem_sim_join()` — embedding similarity cho tất cả pairs (sem_join.py:463)
2. `learn_join_cascade_threshold()` — tìm optimal thresholds (sem_join.py:464-476)
3. High confidence pairs → accept/reject trực tiếp
4. Low confidence pairs → gửi đến oracle LLM

**Map-Search-Filter (MSF) plan:**
1. `map_l1_to_l2()` — dùng sem_map để map left values sang right domain (sem_join.py:483-485)
2. `run_sem_sim_join()` — embedding similarity trên mapped values (sem_join.py:486)
3. `learn_join_cascade_threshold()` — tìm optimal thresholds (sem_join.py:487-498)
4. Routing giống SF plan

**Decision**: Chọn plan có ít LLM calls hơn (sem_join.py:518-527)

### Min cascade size:
Cascade chỉ được dùng khi `num_full_join >= cascade_args.min_join_cascade_size` (sem_join.py:749)

## 6. Input/Output Contract

### Input:
- `other: pd.DataFrame | pd.Series` — DataFrame bên phải (sem_join.py:671)
- `join_instruction: str` — Langex với columns từ cả 2 bên. Hỗ trợ `:left`/`:right` suffix để phân biệt (sem_join.py:700-727)
- `how: str` — Chỉ hỗ trợ "inner" (sem_join.py:696-697)
- `cascade_args: CascadeArgs` — Bao gồm `map_instruction`, `map_examples` cho MSF plan (sem_join.py:766-767)

### Output:
- Joined DataFrame: tất cả rows có join condition = True (sem_join.py:813-817)
- Column rename: nếu trùng tên, thêm `:left`/`:right` suffix (sem_join.py:803-806)
- **return_explanations=True**: Thêm column `explanation_join` (sem_join.py:809)
- **return_stats=True**: Trả về tuple (DataFrame, stats) (sem_join.py:819-820)

### Cascade stats (sem_join.py:318-325):
```python
stats = {
    "join_resolved_by_helper_model": num_helper + num_helper_high_conf_neg,
    "join_helper_positive": num_helper,
    "join_helper_negative": num_helper_high_conf_neg,
    "join_resolved_by_large_model": num_large,
    "optimized_join_cost": join_optimization_cost,
    "total_LM_calls": join_optimization_cost + num_large,
}
```

## 7. Edge Cases

1. **Other là Series không có name**: Raise `ValueError` (sem_join.py:692-693)
2. **Column tồn tại trong cả 2 DataFrames**: Raise `ValueError` khi không có `:left`/`:right` (sem_join.py:716-717)
3. **Left/Right column không tìm thấy**: Assert error (sem_join.py:729-730)
4. **Only inner join**: `NotImplementedError` cho bất kỳ how != "inner" (sem_join.py:696-697)
5. **Cascade threshold learning thất bại**: Default to full join với thresholds (1.0, 0.0) (sem_join.py:600-601)
6. **Empty join result**: Trả về empty DataFrame

## 8. Code Examples

```python
# Basic join
df1.sem_join(df2, "the {article} belongs to the {category}")

# Với disambiguation (:left/:right)
df1.sem_join(df2, "the {name:left} is the same person as {name:right}")

# Với cascade
from lotus.types import CascadeArgs
cascade = CascadeArgs(
    recall_target=0.9,
    precision_target=0.9,
    sampling_percentage=0.1,
    failure_probability=0.2,
)
df1.sem_join(df2, "the {product} matches {item}", cascade_args=cascade)

# Với cascade + custom map instruction
cascade = CascadeArgs(
    recall_target=0.9,
    precision_target=0.9,
    map_instruction="Given {product}, list related items",
)
df1.sem_join(df2, "the {product} matches {item}", cascade_args=cascade)
```

## 9. Assessment

### Điểm mạnh:
- **Join optimizer**: Tự động chọn giữa SF và MSF plans — rất smart
- **Column disambiguation**: Hỗ trợ `:left`/`:right` suffix cho columns trùng tên
- **Cascade stats**: Tracking chi tiết số LLM calls cho mỗi component

### Điểm yếu:
- **O(M*N) complexity**: Không có cascade, chi phí rất lớn cho datasets lớn
- **Only inner join**: Chưa hỗ trợ left/right/outer join (sem_join.py:696-697)
- **Helper LM chưa hỗ trợ**: Comment ghi "Helper model is not supported yet" (sem_join.py:459-460)
- **Safe mode cascade**: Chưa implement cho cascade path (sem_join.py:262-264)
- **Code complexity**: File dài 823 dòng, nhiều nested logic, khó debug
- **sem_filter dependency**: Toàn bộ join logic phụ thuộc vào sem_filter — coupling cao
