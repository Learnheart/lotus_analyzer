# E2 - Extraction Techniques

## Tổng quan

`sem_extract` là operator chính cho structured extraction trong LOTUS. Nó sử dụng LLM để extract JSON-structured data từ unstructured context.

---

## 1. sem_extract Function

**Location**: `sem_ops/sem_extract.py:15-108`

```python
def sem_extract(docs, model, output_cols, extract_quotes=False, ...):
```

- `output_cols: dict[str, str | None]` - mapping tên cột → description
- `extract_quotes: bool` - có extract source quotes không
- Hỗ trợ CoT reasoning qua `strategy` parameter

---

## 2. extract_formatter - Prompt Construction

**Location**: `task_instructions.py:257-321`

### Field description
```python
output_col_names = list(output_cols.keys())                    # :264
output_cols_with_desc = {
    col: col if desc is None else desc                         # :266
    for col, desc in output_cols.items()
}
```

Nếu user không cung cấp description, dùng tên cột làm description.

### System prompt (with quotes) - verbatim từ `task_instructions.py:293-298`:
```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field
and the corresponding full quote for each field in the '{quote_fields}' fields.
Here is a description of each field: {output_cols_with_desc}
The response should be valid JSON format with the following fields: {fields_str}.
```

### System prompt (without quotes) - verbatim từ `task_instructions.py:301-305`:
```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field.
Here is a description of each field: {output_cols_with_desc}
The response should be valid JSON format with the following fields: {fields_str}.
```

### Quote fields
Khi `extract_quotes=True` (`task_instructions.py:270-272`):
```python
quote_fields = [f"{col}_quote" for col in output_col_names]
all_fields = output_col_names + quote_fields
```

---

## 3. Output Format

LLM trả về JSON string. Ví dụ:
```json
{
    "sentiment": "positive",
    "sentiment_quote": "This product is amazing!",
    "rating": "5",
    "rating_quote": "I give it 5 stars"
}
```

### JSON response format
Khi không có CoT reasoning (`sem_extract.py:92`):
```python
lm_output = model(inputs, response_format={"type": "json_object"}, ...)
```
Sử dụng LLM's JSON mode để ensure valid JSON output.

Khi có CoT reasoning (`sem_extract.py:90`):
```python
lm_output = model(inputs, progress_bar_desc=progress_bar_desc)
```
Không dùng JSON mode vì cần reasoning text trước JSON.

---

## 4. Validation và Postprocessing

**Location**: `postprocessors.py:149-179`

```python
def extract_postprocess(llm_answers, model, cot_reasoning=False):
```

### JSON parsing
```python
try:
    output = json.loads(llm_answer)        # postprocessors.py:170-171
except json.JSONDecodeError:
    lotus.logger.info(f"\t Failed to parse: {llm_answer}")
    output = {}                             # postprocessors.py:173-174 - fallback empty dict
```

### Value casting
**Tất cả values đều cast sang string** (`postprocessors.py:176`):
```python
output = {key: str(value) for key, value in output.items()}
```

Điều này nghĩa là:
- LLM trả về `{"count": 5}` → `{"count": "5"}`
- LLM trả về `{"active": true}` → `{"active": "True"}`
- Mọi type information bị mất

### CoT postprocessing
Khi `cot_reasoning=True`, dùng `get_cot_postprocessor` (`postprocessors.py:102-120`):
- Standard: `cot_postprocessor` - tách `Reasoning:` và `Answer:` (`postprocessors.py:12-43`)
- DeepSeek: `deepseek_cot_postprocessor` - tách `<think>` tags (`postprocessors.py:46-93`)
- Answer phần cũng được parse JSON và cast str

---

## 5. DataFrame Integration

**Location**: `sem_extract.py:202-256`

```python
for i, output_dict in enumerate(out.outputs):
    for key, value in output_dict.items():
        if key not in new_df.columns:
            new_df[key] = None                    # sem_extract.py:247
        new_df.loc[indices[i], key] = value       # sem_extract.py:248
```

- Tự động tạo cột mới nếu chưa tồn tại
- Assign values vào đúng row index
- Giữ nguyên DataFrame gốc, chỉ thêm cột mới

---

## 6. Kết luận

Extraction trong LOTUS:
- Schema do user define qua `output_cols` dict
- LLM extract JSON, với optional source quotes
- Validation: json.loads với empty dict fallback
- ALL values cast to string - no type preservation
- JSON mode cho reliable parsing (khi không dùng CoT)
