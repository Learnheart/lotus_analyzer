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

Join 2 DataFrames dua tren dieu kien ngu nghia. Moi cap (left_row, right_row) duoc danh gia True/False boi LLM. Do phuc tap M x N.

**Use cases:**
- Category matching: `df1.sem_join(df2, "the {article} belongs to the {category}")`
- Entity resolution: `df1.sem_join(df2, "the {product:left} is the same as {item:right}")`
- Relationship discovery: `df1.sem_join(df2, "the {person} is mentioned in {document}")`

## 2. Call Stack Trace

```
1. SemJoinDataframe.__call__()                          # sem_join.py:670
2.   lotus.nl_expression.parse_cols(join_instruction)   # sem_join.py:699
3.   Xac dinh left_on va right_on columns               # sem_join.py:700-730
4.   [Neu co examples]: df2multimodal_info(examples)    # sem_join.py:737
5.   [Neu cascade_args != None va du lon]:
5a.    sem_join_cascade()                                # sem_join.py:755-773
5b.      join_optimizer()                                # sem_join.py:242-257
5c.        run_sem_sim_join() cho SF plan                # sem_join.py:463
5d.        learn_join_cascade_threshold() cho SF         # sem_join.py:464-476
5e.        map_l1_to_l2() cho MSF plan                  # sem_join.py:483-485
5f.        run_sem_sim_join() cho MSF plan               # sem_join.py:486
5g.        learn_join_cascade_threshold() cho MSF        # sem_join.py:487-498
5h.        Chon plan re hon                              # sem_join.py:518-527
5i.      Accept high confidence results                  # sem_join.py:267
5j.      sem_filter() cho low confidence                 # sem_join.py:291-301
6.   [Neu khong cascade]:
6a.    sem_join()                                        # sem_join.py:775-791
6b.      Core sem_join():
6c.        df2multimodal_info() cho left va right        # sem_join.py:101-102
6d.        merge_multimodal_info() tao pairs             # sem_join.py:132
6e.        sem_filter() tren tat ca pairs                # sem_join.py:137-147
7.   Build joined DataFrame                             # sem_join.py:798-817
8.   Return DataFrame                                   # sem_join.py:822
```

## 3. Prompt Template (COPY VERBATIM)

sem_join su dung **filter_formatter** (task_instructions.py:87-157) noi bo. Prompt giong het sem_filter:

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

- **Core mechanism**: sem_join goi `sem_filter()` noi bo (sem_join.py:137-147)
- **M x N complexity**: Moi cap (left, right) tao 1 doc roi gui qua sem_filter (sem_join.py:128-135)
- **merge_multimodal_info**: Gop left + right multimodal data (task_instructions.py:382-402)
- **Cascade**: join_optimizer so sanh 2 plans (SF vs MSF) va chon plan re hon (sem_join.py:506-527)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Tat ca M*N pairs gui 1 lan qua sem_filter (sem_join.py:137) |
| Caching | Yes | `@operator_cache` (sem_join.py:669) |
| Cascading | Yes | join_optimizer so sanh SF vs MSF plans (sem_join.py:417-527) |
| Early-termination | No | |
| Sampling | Yes | importance_sampling trong learn_join_cascade_threshold (sem_join.py:566) |
| Safe mode | Partial | Chi estimate cho non-cascade path (sem_join.py:104-120) |

### Join Optimizer chi tiet (sem_join.py:417-527):

**Search-Filter (SF) plan:**
1. `run_sem_sim_join()` — embedding similarity cho tat ca pairs (sem_join.py:463)
2. `learn_join_cascade_threshold()` — tim optimal thresholds (sem_join.py:464-476)
3. High confidence pairs → accept/reject truc tiep
4. Low confidence pairs → gui den oracle LLM

**Map-Search-Filter (MSF) plan:**
1. `map_l1_to_l2()` — dung sem_map de map left values sang right domain (sem_join.py:483-485)
2. `run_sem_sim_join()` — embedding similarity tren mapped values (sem_join.py:486)
3. `learn_join_cascade_threshold()` — tim optimal thresholds (sem_join.py:487-498)
4. Routing giong SF plan

**Decision**: Chon plan co it LLM calls hon (sem_join.py:518-527)

### Min cascade size:
Cascade chi duoc dung khi `num_full_join >= cascade_args.min_join_cascade_size` (sem_join.py:749)

## 6. Input/Output Contract

### Input:
- `other: pd.DataFrame | pd.Series` — DataFrame ben phai (sem_join.py:671)
- `join_instruction: str` — Langex voi columns tu ca 2 bên. Ho tro `:left`/`:right` suffix de phan biet (sem_join.py:700-727)
- `how: str` — Chi ho tro "inner" (sem_join.py:696-697)
- `cascade_args: CascadeArgs` — Bao gom `map_instruction`, `map_examples` cho MSF plan (sem_join.py:766-767)

### Output:
- Joined DataFrame: tat ca rows co join condition = True (sem_join.py:813-817)
- Column rename: neu trung ten, them `:left`/`:right` suffix (sem_join.py:803-806)
- **return_explanations=True**: Them column `explanation_join` (sem_join.py:809)
- **return_stats=True**: Tra ve tuple (DataFrame, stats) (sem_join.py:819-820)

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

1. **Other la Series khong co name**: Raise `ValueError` (sem_join.py:692-693)
2. **Column ton tai trong ca 2 DataFrames**: Raise `ValueError` khi khong co `:left`/`:right` (sem_join.py:716-717)
3. **Left/Right column khong tim thay**: Assert error (sem_join.py:729-730)
4. **Only inner join**: `NotImplementedError` cho bat ky how != "inner" (sem_join.py:696-697)
5. **Cascade threshold learning that bai**: Default to full join voi thresholds (1.0, 0.0) (sem_join.py:600-601)
6. **Empty join result**: Tra ve empty DataFrame

## 8. Code Examples

```python
# Basic join
df1.sem_join(df2, "the {article} belongs to the {category}")

# Voi disambiguation (:left/:right)
df1.sem_join(df2, "the {name:left} is the same person as {name:right}")

# Voi cascade
from lotus.types import CascadeArgs
cascade = CascadeArgs(
    recall_target=0.9,
    precision_target=0.9,
    sampling_percentage=0.1,
    failure_probability=0.2,
)
df1.sem_join(df2, "the {product} matches {item}", cascade_args=cascade)

# Voi cascade + custom map instruction
cascade = CascadeArgs(
    recall_target=0.9,
    precision_target=0.9,
    map_instruction="Given {product}, list related items",
)
df1.sem_join(df2, "the {product} matches {item}", cascade_args=cascade)
```

## 9. Assessment

### Diem manh:
- **Join optimizer**: Tu dong chon giua SF va MSF plans — rat smart
- **Column disambiguation**: Ho tro `:left`/`:right` suffix cho columns trung ten
- **Cascade stats**: Tracking chi tiet so LLM calls cho moi component

### Diem yeu:
- **O(M*N) complexity**: Khong co cascade, chi phi rat lon cho datasets lon
- **Only inner join**: Chua ho tro left/right/outer join (sem_join.py:696-697)
- **Helper LM chua ho tro**: Comment ghi "Helper model is not supported yet" (sem_join.py:459-460)
- **Safe mode cascade**: Chua implement cho cascade path (sem_join.py:262-264)
- **Code complexity**: File dai 823 dong, nhieu nested logic, kho debug
- **sem_filter dependency**: Toan bo join logic phu thuoc vao sem_filter — coupling cao
