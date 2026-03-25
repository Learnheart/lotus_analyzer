# E4 - sem_join Deep Dive

## Tổng quan

`sem_join` là operator phức tạp nhất trong LOTUS - thực hiện semantic join giữa hai bảng bằng LLM. Complexity: O(M x N) LLM calls cho brute-force, có thể giảm đáng kể với cascade optimization.

---

## 1. Core: Cartesian Product LLM Calls

**Location**: `sem_join.py:128-147`

```python
all_docs = []
all_ids1 = []
all_ids2 = []
for id1, i1 in zip(ids1, left_multimodal_data):
    modified_docs = task_instructions.merge_multimodal_info(    # :132
        [i1], right_multimodal_data
    )
    all_docs.extend(modified_docs)                               # :133
    all_ids1.extend([id1] * len(modified_docs))                  # :134
    all_ids2.extend(ids2)                                        # :135

output = sem_filter(                                             # :137
    all_docs, model, user_instruction, ...
)
```

Quá trình:
1. Cho mỗi row trong l1, tạo merged docs với TẤT CẢ rows trong l2 (`sem_join.py:131-135`)
2. `merge_multimodal_info` tạo cartesian product (`task_instructions.py:382-402`)
3. Toàn bộ pairs gửi cho `sem_filter` - reuse filter logic (`sem_join.py:137`)
4. Filter results → join results: `(id1, id2, explanation)` tuples (`sem_join.py:157-163`)

**Cost**: M x N LLM calls (M = len(l1), N = len(l2))

---

## 2. merge_multimodal_info

**Location**: `task_instructions.py:382-402`

```python
def merge_multimodal_info(first, second):
    return [
        {
            "text": f"{first[i]['text']}\n{second[j]['text']}",    # :395-397
            "image": {**first[i]["image"], **second[j]["image"]},  # :398
        }
        for i in range(len(first))
        for j in range(len(second))
    ]
```

- Text: concatenate với newline
- Images: merge dicts (cả hai bảng có images)
- Cartesian product: len(first) x len(second) outputs

---

## 3. Cascade Optimization: join_optimizer

**Location**: `sem_join.py:417-527`

```python
def join_optimizer(l1, l2, col1_label, col2_label, model, user_instruction,
                   cascade_args, ...):
```

### So sánh hai Join Plans

#### Plan 1: Search-Filter (SF)
```python
sf_helper_join = run_sem_sim_join(l1, l2, col1_label, col2_label)    # :463
sf_t_pos, sf_t_neg, sf_learn_cost = learn_join_cascade_threshold(    # :464-476
    sf_helper_join, ...
)
sf_high_conf = sf_helper_join[sf_helper_join["_scores"] >= sf_t_pos]  # :477
sf_low_conf = sf_helper_join[...between thresholds...]                 # :479
sf_cost = len(sf_low_conf)                                            # :480
```

1. Chạy sem_sim_join (embedding similarity) cho tất cả pairs
2. Learn thresholds từ sample
3. High confidence → accept without LLM
4. Low confidence → send to LLM

#### Plan 2: Map-Search-Filter (MSF)
```python
mapped_l1, mapped_col1_label = map_l1_to_l2(                        # :483-485
    l1, col1_label, col2_label, map_instruction, map_examples
)
msf_helper_join = run_sem_sim_join(mapped_l1, l2, ...)               # :486
msf_cost = len(msf_low_conf)                                         # :503
msf_learn_cost += len(l1)  # cost from map                           # :504
```

1. sem_map l1 sang l2's domain (ví dụ: articles → categories)
2. Chạy sem_sim_join trên mapped data
3. Same threshold logic as SF

#### Plan Selection
```python
if sf_cost < msf_cost:                                               # :518
    lotus.logger.info("Proceeding with Search-Filter")               # :519
    return sf_high_conf, sf_low_conf, ...                            # :522
else:
    lotus.logger.info("Proceeding with Map-Search-Filter")           # :524
    return msf_high_conf, msf_low_conf, ...                          # :527
```

Chọn plan có ít LLM calls cho low-confidence pairs.

---

## 4. map_l1_to_l2

**Location**: `sem_join.py:369-414`

```python
def map_l1_to_l2(l1, col1_label, col2_label, map_instruction=None, map_examples=None):
```

Default instruction (`sem_join.py:403`):
```
"Given {real_left_on}, identify the most relevant {real_right_on}.
 Always write your answer as a list of 2-10 comma-separated {real_right_on}."
```

Ví dụ: articles → categories mapping:
- Input: "Machine learning tutorial"
- Output: "Computer Science, AI, Machine Learning"

---

## 5. run_sem_sim_join

**Location**: `sem_join.py:336-366`

```python
def run_sem_sim_join(l1, l2, col1_label, col2_label):
    l2_df = l2_df.sem_index(col2_label, ...)     # :358
    out = l1_df.sem_sim_join(l2_df, ..., K=K)    # :362
    out["_scores"] = calibrate_sem_sim_join(...)  # :365
```

Tạo embedding index cho l2, rồi similarity join l1 với l2.

---

## 6. Threshold Learning cho Join

**Location**: `sem_join.py:530-603`

```python
def learn_join_cascade_threshold(helper_join, col1_label, col2_label, model, ...):
    sample_indices, correction_factors = importance_sampling(helper_scores, cascade_args)  # :566
    # ... sample, run oracle, learn thresholds ...
    (pos_threshold, neg_threshold), _ = learn_cascade_thresholds(...)                      # :589
```

Giống filter cascade nhưng applied cho join scores.

---

## 7. Performance Summary

| Scenario | LLM Calls | Notes |
|---|---|---|
| No cascade | M x N | Full cartesian product |
| SF cascade | sf_low_conf + learn_cost | Embedding similarity pre-filter |
| MSF cascade | msf_low_conf + map_cost + learn_cost | Map + similarity pre-filter |

Cascade có thể giảm calls dramatically khi:
- Nhiều pairs có high similarity score (→ high confidence accept)
- Nhiều pairs có low similarity score (→ high confidence reject)
- Chỉ ambiguous pairs cần LLM verification

---

## 8. Kết luận

sem_join là operator tốn nhất trong LOTUS (O(M x N)), nhưng cascade optimization có thể giảm đáng kể:
- Automatic plan selection (SF vs MSF)
- Statistical threshold learning
- Only ambiguous pairs sent to LLM
