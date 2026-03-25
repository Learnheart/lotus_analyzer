# SEM_EXTRACT — `sem_extract`

## Metadata
- **File**: `lotus/sem_ops/sem_extract.py`
- **Accessor line**: 111 (`@pd.api.extensions.register_dataframe_accessor("sem_extract")`)
- **Core function line**: 15 (`def sem_extract(...)`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_extract.py:216-219)

## 1. Purpose & Use Cases

Trích xuất thông tin có cấu trúc từ documents. Output là JSON với các fields được định nghĩa trước. Hỗ trợ extract quotes để attribution.

**Use cases:**
- Entity extraction: `df.sem_extract(["text"], {"person": "name of person", "age": "age in years"})`
- Attribute extraction: `df.sem_extract(["review"], {"sentiment": "positive/negative", "rating": "1-5"})`
- Source attribution: `df.sem_extract(["doc"], {"claim": None}, extract_quotes=True)`

## 2. Call Stack Trace

```
1. SemExtractDataFrame.__call__()                       # sem_extract.py:202
2.   Column validation                                  # sem_extract.py:222-224
3.   df2multimodal_info(df, input_cols)                 # sem_extract.py:226
4.   sem_extract(multimodal_data, lm, output_cols, ...) # sem_extract.py:228-238
5.   Core sem_extract():
5a.    extract_formatter(model, doc, output_cols, ...)   # sem_extract.py:76
5b.    [Nếu CoT]: model(inputs) không JSON format        # sem_extract.py:89-90
5c.    [Else]: model(inputs, response_format=json_object) # sem_extract.py:92
5d.    postprocessor(lm_output.outputs, model, cot)      # sem_extract.py:97
6.   Populate DataFrame với extracted values             # sem_extract.py:242-248
7.   Return DataFrame                                   # sem_extract.py:256
```

## 3. Prompt Template (COPY VERBATIM)

### System instruction — without quotes (task_instructions.py:301-306):
```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field.
Here is a description of each field: {output_cols_with_desc}
The response should be valid JSON format with the following fields: {fields_str}.
```

### System instruction — with quotes (task_instructions.py:293-299):
```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field and the corresponding full quote for each field in the '{quote_fields}' fields.
Here is a description of each field: {output_cols_with_desc}
The response should be valid JSON format with the following fields: {fields_str}.
```

### CoT addition (task_instructions.py:277-288):
```
Let's think step by step. Use the following format to provide your answer:
        Reasoning:
<Your reasoning here. Think through each extraction step by step.>

Answer: <Your answer here. Provide the JSON response with fields: {fields_str}>
```

### User message (task_instructions.py:313-314):
```
Context:
{serialized row data}
```

## 4. LLM Interaction

- **Model**: `lotus.settings.lm` (sem_extract.py:229)
- **JSON response format**: Dùng `response_format={"type": "json_object"}` khi không CoT (sem_extract.py:92)
- **CoT conflict**: Khi CoT enabled, không dùng JSON response format vì nó ngăn reasoning text (sem_extract.py:88-90)
- **Postprocessing**: `extract_postprocess()` (postprocessors.py:149-179)
  - Parse JSON từ LLM output (postprocessors.py:171)
  - Convert tất cả values thành string (postprocessors.py:176)
  - Khi parse thất bại: trả về empty dict `{}` (postprocessors.py:174)

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | Tất cả inputs gửi 1 lần qua model (sem_extract.py:90-92) |
| Caching | Yes | `@operator_cache` (sem_extract.py:201) |
| Cascading | No | |
| Early-termination | No | |
| Sampling | No | |
| Safe mode | Yes | Estimate cost (sem_extract.py:82-85) |

## 6. Input/Output Contract

### Input:
- `input_cols: list[str]` — Columns làm input cho extraction (sem_extract.py:203)
- `output_cols: dict[str, str | None]` — {field_name: description} (sem_extract.py:204)
- `extract_quotes: bool` — Trích quote từ source (sem_extract.py:205, default=False)
- `postprocessor: Callable` — Custom postprocessor (sem_extract.py:206-209)
- `strategy: ReasoningStrategy` — None, COT, ZS_COT (sem_extract.py:214)

### Output:
- DataFrame gốc + new columns cho mỗi key trong output_cols (sem_extract.py:242-248)
- Nếu `extract_quotes=True`: thêm columns `{field}_quote` (task_instructions.py:271)
- **return_raw_outputs=True**: Thêm column `raw_output` (sem_extract.py:250-251)
- **return_explanations=True**: Thêm column `explanation` (sem_extract.py:253-254)

### Đặc biệt về populate logic (sem_extract.py:242-248):
```python
for i, output_dict in enumerate(out.outputs):
    for key, value in output_dict.items():
        if key not in new_df.columns:
            new_df[key] = None  # Tạo column mới nếu chưa có
        new_df.loc[indices[i], key] = value  # Set value theo index
```

## 7. Edge Cases

1. **Column không tồn tại**: Raise `ValueError` (sem_extract.py:223-224)
2. **LM chưa configure**: Raise `ValueError` (sem_extract.py:216-219)
3. **JSON parse thất bại**: Trả về empty dict `{}` (postprocessors.py:173-174)
4. **output_cols description là None**: Dùng key name làm description (task_instructions.py:266)
5. **CoT + JSON format conflict**: CoT disabled JSON response_format (sem_extract.py:88-92)
6. **Extra keys in output**: LLM có thể trả về keys ngoài output_cols — tất cả đều được thêm vào DataFrame (sem_extract.py:245-248)
7. **Missing keys in output**: Nếu LLM không trả về 1 key, column đó sẽ là None

## 8. Code Examples

```python
# Basic extraction
df.sem_extract(
    ["text"],
    {"sentiment": "positive/negative/neutral", "confidence": "0-1 scale"}
)

# Với quote extraction
df.sem_extract(
    ["document"],
    {"main_claim": "primary argument", "evidence": "supporting evidence"},
    extract_quotes=True
)

# Với CoT reasoning
df.sem_extract(
    ["text"],
    {"category": "product category"},
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True
)

# Multiple input columns
df.sem_extract(
    ["title", "abstract"],
    {"topic": "research topic", "methodology": "research method"}
)
```

## 9. Assessment

### Điểm mạnh:
- **Structured output**: JSON format đảm bảo output có cấu trúc
- **Quote extraction**: extract_quotes rất hữu ích cho attribution/explainability
- **JSON response format**: Dùng `response_format=json_object` đảm bảo valid JSON
- **Flexible schema**: output_cols có thể có descriptions hoặc None

### Điểm yếu:
- **All values are strings**: `str(value)` convert tất cả — mất type information (postprocessors.py:176)
- **Extra keys accepted**: LLM có thể trả về keys bất kỳ, không có validation (sem_extract.py:245)
- **No schema validation**: Không kiểm tra output có đúng schema hay không
- **CoT + JSON conflict**: Khi dùng CoT, không có JSON format guarantee
- **Khác với sem_map**: Dùng `input_cols` list thay vì langex — inconsistent API
