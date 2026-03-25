# SEM_MAP — `sem_map`

## Metadata
- **File**: `lotus/sem_ops/sem_map.py`
- **Accessor line**: 121 (`@pd.api.extensions.register_dataframe_accessor("sem_map")`)
- **Core function line**: 14 (`def sem_map(...)`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_map.py:229)

## 1. Purpose & Use Cases

Áp dụng một phép biến đổi ngôn ngữ tự nhiên lên mỗi row của DataFrame. Kết quả là một column mới chứa output của LLM.

**Use cases:**
- Sentiment labeling: `df.sem_map("Label the sentiment of {text} as positive/negative")`
- Summarization: `df.sem_map("Summarize {document} in one sentence")`
- Classification: `df.sem_map("Classify the {email} as spam or not-spam")`
- Translation: `df.sem_map("Translate {text} to Vietnamese")`

## 2. Call Stack Trace

```
1. SemMapDataframe.__call__()                          # sem_map.py:215
2.   lotus.nl_expression.parse_cols(user_instruction)  # sem_map.py:234
3.   Column validation loop                            # sem_map.py:237-239
4.   task_instructions.df2multimodal_info(df, col_li)  # sem_map.py:241
5.   lotus.nl_expression.nle2str(user_instruction)     # sem_map.py:242
6.   [Nếu có examples]:
6a.    df2multimodal_info(examples, col_li)             # sem_map.py:250
6b.    examples["Answer"].tolist()                      # sem_map.py:251
6c.    [Nếu COT/ZS_COT]: examples["Reasoning"].tolist() # sem_map.py:255
7.   sem_map(multimodal_data, lm, ...)                 # sem_map.py:257-270
8.   Core sem_map():
8a.    map_formatter() cho mỗi doc                      # sem_map.py:81-90
8b.    model(inputs, ...)                               # sem_map.py:102
8c.    postprocessor(lm_output.outputs, ...)            # sem_map.py:105-107
9.   new_df[suffix] = output.outputs                   # sem_map.py:273
10.  Return DataFrame                                  # sem_map.py:279
```

## 3. Prompt Template (COPY VERBATIM)

### Default system instruction (task_instructions.py:223-226):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.
```

### ZS-CoT system instruction (task_instructions.py:200-203):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.First give your reasoning. Then you MUST end your output with "Answer: your answer"
```

### CoT system instruction (task_instructions.py:168-172):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.You must give your reasoning and then your final answer
```

### User message format (task_instructions.py:68-84):
```
Context:
{serialized row data}

Instruction: {user_instruction}
```

### Few-shot example format (task_instructions.py:241-246):
```
[User]: Context: {example_data}  Instruction: {instruction}
[Assistant]: {answer}
```

### CoT few-shot format (task_instructions.py:181-189):
```
[User]: Context: {example_data}  Instruction: {instruction}
[Assistant]: Reasoning:
{cot_reasoning}

Answer: {answer}
```

## 4. LLM Interaction

- **Model**: `lotus.settings.lm` (sem_map.py:259)
- **Call**: `model(inputs, progress_bar_desc=..., **model_kwargs)` (sem_map.py:102)
- **Custom system_prompt**: Có thể truyền custom system prompt qua parameter (sem_map.py:18, 223-226)
- **Extra kwargs**: Truyền thêm bất kỳ keyword arg nào cho model (sem_map.py:26, 269)
- **Postprocessing**: `map_postprocess()` (postprocessors.py:123-146)
  - Nếu CoT: tách Reasoning và Answer (postprocessors.py:139-141)
  - Nếu không CoT: output = raw LLM output (postprocessors.py:143)

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | Toàn bộ inputs gửi 1 lần qua `model(inputs)` (sem_map.py:102) |
| Caching | Yes | `@operator_cache` decorator (sem_map.py:214) |
| Cascading | No | Không hỗ trợ cascade |
| Early-termination | No | |
| Sampling | No | |
| Safe mode | Yes | Ước tính cost (sem_map.py:96-99) |

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Langex expression với `{column}` placeholders
- `system_prompt: str | None` — Custom system prompt (sem_map.py:218)
- `postprocessor: Callable` — Custom postprocessor function (sem_map.py:219)
- `suffix: str` — Tên column output, default `"_map"` (sem_map.py:222)
- `examples: pd.DataFrame` — Phải có column "Answer", optional "Reasoning" (sem_map.py:248-255)
- `strategy: ReasoningStrategy` — None, COT, ZS_COT (sem_map.py:224)
- `**model_kwargs` — Truyền thêm cho model (sem_map.py:227)

### Output:
- DataFrame gốc + column `suffix` (default `_map`) chứa LLM output strings (sem_map.py:272-273)
- **return_explanations=True**: Thêm column `explanation_map` (sem_map.py:274-275)
- **return_raw_outputs=True**: Thêm column `raw_output_map` (sem_map.py:276-277)

## 7. Edge Cases

1. **Column không tồn tại**: Raise `ValueError` (sem_map.py:238-239)
2. **LM chưa configure**: Raise `ValueError` (sem_map.py:229-232)
3. **Examples không có "Answer" column**: Assert error (sem_map.py:249)
4. **COT/ZS_COT với examples**: Tự động set `return_explanations=True` và lấy "Reasoning" column (sem_map.py:253-255) — có thể raise error nếu "Reasoning" column không tồn tại
5. **Custom postprocessor**: Phải có signature `(list[str], LM, bool) -> SemanticMapPostprocessOutput` (sem_map.py:19)

## 8. Code Examples

```python
# Basic mapping
df.sem_map("Label the sentiment of {text} as positive/negative")

# Với custom system prompt
df.sem_map(
    "Classify the {document}",
    system_prompt="You are an expert document classifier."
)

# Với few-shot examples
examples = pd.DataFrame({
    "text": ["Great product!", "Terrible service"],
    "Answer": ["positive", "negative"]
})
df.sem_map("Classify sentiment of {text}", examples=examples)

# Với ZS-CoT
df.sem_map(
    "What is the main topic of {text}?",
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True
)

# Custom suffix
df.sem_map("Translate {text} to French", suffix="_french")
```

## 9. Assessment

### Điểm mạnh:
- **Flexibility**: Custom system_prompt, custom postprocessor, **model_kwargs — rất linh hoạt
- **Clean API**: Đơn giản, trực quan, dùng như SQL SELECT + transform
- **Few-shot support**: Hỗ trợ đầy đủ với CoT reasoning examples

### Điểm yếu:
- **Không có cascade**: Không như sem_filter, sem_map không hỗ trợ cascade optimization
- **No type validation**: Output là string, không có validation output format
- **COT examples bug potential**: Khi `strategy=COT` mà examples không có "Reasoning" column, sẽ raise KeyError (sem_map.py:255) — không có error message rõ ràng
- **Suffix collision**: Không kiểm tra xem suffix đã tồn tại trong DataFrame chưa (sem_map.py:273), có thể overwrite column
