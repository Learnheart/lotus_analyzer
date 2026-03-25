# SEM_EXTRACT — `sem_extract`

## Metadata
- **File**: `lotus/sem_ops/sem_extract.py`
- **Accessor line**: 111 (`@pd.api.extensions.register_dataframe_accessor("sem_extract")`)
- **Core function line**: 15 (`def sem_extract(...)`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_extract.py:216-219)

## 1. Purpose & Use Cases

Trich xuat thong tin co cau truc tu documents. Output la JSON voi cac fields duoc dinh nghia truoc. Ho tro extract quotes de attribution.

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
5b.    [Neu CoT]: model(inputs) khong JSON format        # sem_extract.py:89-90
5c.    [Else]: model(inputs, response_format=json_object) # sem_extract.py:92
5d.    postprocessor(lm_output.outputs, model, cot)      # sem_extract.py:97
6.   Populate DataFrame voi extracted values             # sem_extract.py:242-248
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
- **JSON response format**: Dung `response_format={"type": "json_object"}` khi khong CoT (sem_extract.py:92)
- **CoT conflict**: Khi CoT enabled, khong dung JSON response format vi no ngan reasoning text (sem_extract.py:88-90)
- **Postprocessing**: `extract_postprocess()` (postprocessors.py:149-179)
  - Parse JSON tu LLM output (postprocessors.py:171)
  - Convert tat ca values thanh string (postprocessors.py:176)
  - Khi parse that bai: tra ve empty dict `{}` (postprocessors.py:174)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Tat ca inputs gui 1 lan qua model (sem_extract.py:90-92) |
| Caching | Yes | `@operator_cache` (sem_extract.py:201) |
| Cascading | No | |
| Early-termination | No | |
| Sampling | No | |
| Safe mode | Yes | Estimate cost (sem_extract.py:82-85) |

## 6. Input/Output Contract

### Input:
- `input_cols: list[str]` — Columns lam input cho extraction (sem_extract.py:203)
- `output_cols: dict[str, str | None]` — {field_name: description} (sem_extract.py:204)
- `extract_quotes: bool` — Trich quote tu source (sem_extract.py:205, default=False)
- `postprocessor: Callable` — Custom postprocessor (sem_extract.py:206-209)
- `strategy: ReasoningStrategy` — None, COT, ZS_COT (sem_extract.py:214)

### Output:
- DataFrame goc + new columns cho moi key trong output_cols (sem_extract.py:242-248)
- Neu `extract_quotes=True`: them columns `{field}_quote` (task_instructions.py:271)
- **return_raw_outputs=True**: Them column `raw_output` (sem_extract.py:250-251)
- **return_explanations=True**: Them column `explanation` (sem_extract.py:253-254)

### Dac biet ve populate logic (sem_extract.py:242-248):
```python
for i, output_dict in enumerate(out.outputs):
    for key, value in output_dict.items():
        if key not in new_df.columns:
            new_df[key] = None  # Tao column moi neu chua co
        new_df.loc[indices[i], key] = value  # Set value theo index
```

## 7. Edge Cases

1. **Column khong ton tai**: Raise `ValueError` (sem_extract.py:223-224)
2. **LM chua configure**: Raise `ValueError` (sem_extract.py:216-219)
3. **JSON parse that bai**: Tra ve empty dict `{}` (postprocessors.py:173-174)
4. **output_cols description la None**: Dung key name lam description (task_instructions.py:266)
5. **CoT + JSON format conflict**: CoT disabled JSON response_format (sem_extract.py:88-92)
6. **Extra keys in output**: LLM co the tra ve keys ngoai output_cols — tat ca deu duoc them vao DataFrame (sem_extract.py:245-248)
7. **Missing keys in output**: Neu LLM khong tra ve 1 key, column do se la None

## 8. Code Examples

```python
# Basic extraction
df.sem_extract(
    ["text"],
    {"sentiment": "positive/negative/neutral", "confidence": "0-1 scale"}
)

# Voi quote extraction
df.sem_extract(
    ["document"],
    {"main_claim": "primary argument", "evidence": "supporting evidence"},
    extract_quotes=True
)

# Voi CoT reasoning
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

### Diem manh:
- **Structured output**: JSON format dam bao output co cau truc
- **Quote extraction**: extract_quotes rat huu ich cho attribution/explainability
- **JSON response format**: Dung `response_format=json_object` dam bao valid JSON
- **Flexible schema**: output_cols co the co descriptions hoac None

### Diem yeu:
- **All values are strings**: `str(value)` convert tat ca — mat type information (postprocessors.py:176)
- **Extra keys accepted**: LLM co the tra ve keys bat ky, khong co validation (sem_extract.py:245)
- **No schema validation**: Khong kiem tra output co dung schema hay khong
- **CoT + JSON conflict**: Khi dung CoT, khong co JSON format guarantee
- **Khac voi sem_map**: Dung `input_cols` list thay vi langex — inconsistent API
