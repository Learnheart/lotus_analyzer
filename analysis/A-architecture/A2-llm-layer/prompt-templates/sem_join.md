# Prompt Template: sem_join

## Su dung lai filter_formatter

`sem_join` KHONG co prompt template rieng. No reuse `filter_formatter` tu `task_instructions.py:87`, giong het voi `sem_filter`.

## Cach hoat dong

### sem_join function (sem_join.py:16)

```python
def sem_join(l1, l2, ids1, ids2, col1_label, col2_label, model, user_instruction, ...):
    left_multimodal_data = task_instructions.df2multimodal_info(l1.to_frame(col1_label), [col1_label])
    right_multimodal_data = task_instructions.df2multimodal_info(l2.to_frame(col2_label), [col2_label])

    all_docs = []
    for id1, i1 in zip(ids1, left_multimodal_data):
        modified_docs = task_instructions.merge_multimodal_info([i1], right_multimodal_data)
        all_docs.extend(modified_docs)

    output = sem_filter(
        all_docs,
        model,
        user_instruction,
        ...
    )
```

### merge_multimodal_info (task_instructions.py:382)

Ket hop 2 multimodal dicts thanh 1:
```python
{
    "text": f"{first[i]['text']}\n{second[j]['text']}",
    "image": {**first[i]["image"], **second[j]["image"]},
}
```

### Ket qua prompt cho moi cap (left_row, right_row)

```
Context:
[Article]: <<Machine learning tutorial>>
[Category]: <<Computer Science>>

Claim: the Article belongs to the Category
```

Day la chinh xac cung prompt format nhu `sem_filter`, nhung context chua du lieu tu CA 2 DataFrames.

## Safe mode estimation (sem_join.py:104-120)

```python
if safe_mode:
    sample_docs = task_instructions.merge_multimodal_info([left_multimodal_data[0]], right_multimodal_data)
    estimated_tokens_per_call = model.count_tokens(
        lotus.templates.task_instructions.filter_formatter(
            model, sample_docs[0], user_instruction, ...
        )
    )
    estimated_total_calls = len(l1) * len(l2)
    estimated_total_cost = estimated_tokens_per_call * estimated_total_calls
```

Join co so LLM calls = `len(left) * len(right)` (cross product), co the rat lon.

## Join Cascade

Khi `cascade_args` duoc cung cap, join su dung `sem_sim_join` de lam proxy model va chi gui low-confidence pairs den LLM (sem_join.py:746-773). Xem `sem_join.py:417` (`join_optimizer`) de hieu chi tiet.

## Phan tich

1. **Reuse pattern**: Thay vi viet prompt rieng, `sem_join` convert bai toan join thanh bai toan filter bang cach merge context tu 2 DataFrames.

2. **Cross product cost**: O(n*m) LLM calls. Cascade giup giam so calls bang cach dung embedding similarity de loc truoc.

3. **Column disambiguation**: `:left` va `:right` suffixes cho phep user chi dinh column nao thuoc DataFrame nao khi 2 DataFrames co cung column name (sem_join.py:703-707).
