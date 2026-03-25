# E1 - Mixed Schema Design

## Tổng quan

LOTUS không phân biệt giữa structured và unstructured columns khi serialize cho LLM. Tất cả đều được convert thành text và gửi trong cùng một prompt.

---

## 1. Mixed Schema Example

Cho table:
```
| price (float) | description (text)           |
|---------------|------------------------------|
| 29.99         | Great wireless headphones    |
| 149.99        | Premium noise-cancelling set |
```

Khi serialize qua `df2text` (`task_instructions.py:329`), output cho mỗi row:
```
[Price]: «29.99»
[Description]: «Great wireless headphones»
```

Cả `price` (float) lẫn `description` (text) đều được format **giống hệt nhau** - chỉ khác giá trị bên trong `« »`.

---

## 2. Langex với mixed predicates

Khi user viết langex chứa cả structured và unstructured conditions:

```python
df.sem_filter("the {description} matches wireless headphones and {price} < 100")
```

### Quá trình xử lý:

1. **Parse columns**: `nl_expression.parse_cols()` (`nl_expression.py:4-14`) trích xuất `["description", "price"]`

2. **Serialize**: `df2multimodal_info(df, ["description", "price"])` (`task_instructions.py:364-379`) tạo context text cho mỗi row

3. **Format prompt**: `filter_formatter` (`task_instructions.py:87-157`) tạo prompt:
   ```
   System: Your job is to determine whether the claim is true...
   User: Context:
   [Description]: «Great wireless headphones»
   [Price]: «29.99»

   Claim: the Description matches wireless headphones and Price < 100
   ```

4. **LLM decides**: LLM nhận toàn bộ claim bao gồm CẢ điều kiện structured (`price < 100`) và unstructured (`description matches`). KHÔNG có structured predicate extraction.

---

## 3. Không có Predicate Pushdown

Điểm quan trọng: LOTUS **KHÔNG** tách structured predicates ra khỏi unstructured predicates. Toàn bộ langex được gửi cho LLM.

- Không có query optimizer tách `price < 100` thành pandas filter trước
- Không có hybrid execution (SQL cho structured, LLM cho unstructured)
- LLM phải xử lý cả so sánh số (`< 100`) lẫn semantic matching

Điều này ảnh hưởng đến:
- **Accuracy**: LLM có thể sai khi so sánh số (ví dụ: "99.99 < 100" có thể bị miss)
- **Cost**: Mỗi row đều cần LLM call, kể cả khi structured predicate đã loại row đó
- **Latency**: Không thể filter nhanh bằng pandas trước khi gọi LLM

---

## 4. sem_join với mixed columns

`sem_join` xử lý mixed schemas bằng cách merge multimodal info từ cả hai bảng:

```python
# sem_join.py:131-135
for id1, i1 in zip(ids1, left_multimodal_data):
    modified_docs = task_instructions.merge_multimodal_info([i1], right_multimodal_data)
```

`merge_multimodal_info` (`task_instructions.py:382-402`) kết hợp text và image data:
```python
"text": f"{first[i]['text']}\n{second[j]['text']}"
"image": {**first[i]["image"], **second[j]["image"]}
```

TẤT CẢ columns referenced trong langex đều được include trong context cho LLM - không có selection/optimization dựa trên column type.

---

## 5. Implications cho mixed schema

- LLM phải đồng thời hiểu numeric comparisons VÀ semantic matching
- Không có type-aware optimization
- Trade-off: flexibility (mọi kiểu query) vs efficiency (LLM xử lý mọi thứ)
