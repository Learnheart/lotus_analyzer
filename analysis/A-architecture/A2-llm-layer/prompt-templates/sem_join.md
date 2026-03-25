# Prompt Template: sem_join

## Sử dụng lại filter_formatter

`sem_join` KHÔNG có prompt template riêng. Nó reuse `filter_formatter` từ `task_instructions.py:87`, giống hệt với `sem_filter`.

## Cách hoạt động

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

Kết hợp 2 multimodal dicts thành 1:
```python
{
    "text": f"{first[i]['text']}\n{second[j]['text']}",
    "image": {**first[i]["image"], **second[j]["image"]},
}
```

### Kết quả prompt cho mỗi cặp (left_row, right_row)

```
Context:
[Article]: <<Machine learning tutorial>>
[Category]: <<Computer Science>>

Claim: the Article belongs to the Category
```

Đây là chính xác cùng prompt format như `sem_filter`, nhưng context chứa dữ liệu từ CẢ 2 DataFrames.

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

Join có số LLM calls = `len(left) * len(right)` (cross product), có thể rất lớn.

## Join Cascade

Khi `cascade_args` được cung cấp, join sử dụng `sem_sim_join` để làm proxy model và chỉ gửi low-confidence pairs đến LLM (sem_join.py:746-773). Xem `sem_join.py:417` (`join_optimizer`) để hiểu chi tiết.

## Phân tích

1. **Reuse pattern**: Thay vì viết prompt riêng, `sem_join` convert bài toán join thành bài toán filter bằng cách merge context từ 2 DataFrames.

2. **Cross product cost**: O(n*m) LLM calls. Cascade giúp giảm số calls bằng cách dùng embedding similarity để lọc trước.

3. **Column disambiguation**: `:left` và `:right` suffixes cho phép user chỉ định column nào thuộc DataFrame nào khi 2 DataFrames có cùng column name (sem_join.py:703-707).
