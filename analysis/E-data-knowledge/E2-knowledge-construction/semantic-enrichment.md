# E2 - Semantic Enrichment

## Tổng quan

LOTUS cho phép enrichment dữ liệu thông qua các semantic operators, tạo ra cột mới từ LLM reasoning. Chuỗi enrichment có thể chain nhiều operators để tạo knowledge ngày càng phong phú.

---

## 1. sem_map - Knowledge Creation

`sem_map` tạo cột mới bằng cách áp dụng LLM instruction lên mỗi row:

```python
df.sem_map("extract sentiment of {review}")
```

Mỗi row được serialize → gửi cho LLM → nhận output string → lưu vào cột mới.

### Prompt format

`map_formatter` (`task_instructions.py:213-254`):
```
System: The user will provide an instruction and some relevant context.
        Your job is to answer the user's instruction given the context.
User: Context:
      [Review]: «This product is amazing!»

      Instruction: extract sentiment of Review
```

Output là raw string từ LLM, lưu vào cột mới với suffix (default `"_output"`).

---

## 2. sem_extract - Structured Column Creation

`sem_extract` tạo nhiều cột structured từ unstructured text:

```python
df.sem_extract(
    input_cols=["review"],
    output_cols={"sentiment": "positive/negative/neutral", "rating": "1-5 scale"}
)
```

### Quá trình

1. Format prompt qua `extract_formatter` (`task_instructions.py:257-321`)
2. LLM trả về JSON: `{"sentiment": "positive", "rating": "5"}`
3. `extract_postprocess` parse JSON (`postprocessors.py:149-179`):
   - `json.loads(llm_answer)` (`postprocessors.py:170-171`)
   - Cast all values: `{key: str(value) for key, value in output.items()}` (`postprocessors.py:176`)
4. Assign vào DataFrame:
   ```python
   for key, value in output_dict.items():
       new_df.loc[indices[i], key] = value  # sem_extract.py:248
   ```

### extract_quotes option

Khi `extract_quotes=True` (`task_instructions.py:292-298`):
- Thêm `_quote` fields vào output: `quote_fields = [f"{col}_quote" for col in output_col_names]` (`task_instructions.py:271`)
- LLM phải provide source quote cho mỗi extracted value

---

## 3. Chain of Enrichment

LOTUS operators có thể chain để tạo enrichment pipeline:

```python
df = (
    df
    .sem_map("extract key topics from {review}")          # Step 1: create topics column
    .sem_extract(
        ["review"],
        {"sentiment": "positive/negative", "entity": "product name"}
    )                                                       # Step 2: extract structured fields
    .sem_filter("the {sentiment} is positive")             # Step 3: filter enriched data
)
```

Mỗi step:
1. Nhận DataFrame từ step trước
2. Thêm cột mới hoặc filter rows
3. Trả về DataFrame mới cho step sau

---

## 4. Derived Columns cho Subsequent Search

Các cột được tạo bởi sem_map/sem_extract có thể được index cho search:

```python
# Enrichment
df = df.sem_map("summarize {long_document}", suffix="_summary")

# Index derived column
df = df.sem_index("_summary", "summary_index")

# Search on derived column
results = df.sem_search("_summary", "machine learning applications", K=5)
```

Điều này cho phép:
- Tạo summaries → index summaries → search trên summaries (nhanh hơn search trên full documents)
- Extract entities → index entities → search trên entities

---

## 5. Implications

- **Pro**: Flexible - bất kỳ LLM-derivable information nào đều có thể trở thành column
- **Pro**: Composable - chain operators tạo complex pipelines
- **Con**: Costly - mỗi enrichment step = N LLM calls (N = số rows)
- **Con**: Error propagation - errors trong step sớm ảnh hưởng tất cả steps sau
- **Con**: No rollback - không có transaction mechanism cho failed enrichment
