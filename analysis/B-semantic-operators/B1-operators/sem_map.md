# SEM_MAP — `sem_map`

## Metadata
- **File**: `lotus/sem_ops/sem_map.py`
- **Accessor line**: 121 (`@pd.api.extensions.register_dataframe_accessor("sem_map")`)
- **Core function line**: 14 (`def sem_map(...)`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_map.py:229)

## 1. Purpose & Use Cases

Ap dung mot phep bien doi ngon ngu tu nhien len moi row cua DataFrame. Ket qua la mot column moi chua output cua LLM.

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
6.   [Neu co examples]:
6a.    df2multimodal_info(examples, col_li)             # sem_map.py:250
6b.    examples["Answer"].tolist()                      # sem_map.py:251
6c.    [Neu COT/ZS_COT]: examples["Reasoning"].tolist() # sem_map.py:255
7.   sem_map(multimodal_data, lm, ...)                 # sem_map.py:257-270
8.   Core sem_map():
8a.    map_formatter() cho moi doc                      # sem_map.py:81-90
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
- **Custom system_prompt**: Co the truyen custom system prompt qua parameter (sem_map.py:18, 223-226)
- **Extra kwargs**: Truyen them bat ky keyword arg nao cho model (sem_map.py:26, 269)
- **Postprocessing**: `map_postprocess()` (postprocessors.py:123-146)
  - Neu CoT: tach Reasoning va Answer (postprocessors.py:139-141)
  - Neu khong CoT: output = raw LLM output (postprocessors.py:143)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Toan bo inputs gui 1 lan qua `model(inputs)` (sem_map.py:102) |
| Caching | Yes | `@operator_cache` decorator (sem_map.py:214) |
| Cascading | No | Khong ho tro cascade |
| Early-termination | No | |
| Sampling | No | |
| Safe mode | Yes | Uoc tinh cost (sem_map.py:96-99) |

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Langex expression voi `{column}` placeholders
- `system_prompt: str | None` — Custom system prompt (sem_map.py:218)
- `postprocessor: Callable` — Custom postprocessor function (sem_map.py:219)
- `suffix: str` — Ten column output, default `"_map"` (sem_map.py:222)
- `examples: pd.DataFrame` — Phai co column "Answer", optional "Reasoning" (sem_map.py:248-255)
- `strategy: ReasoningStrategy` — None, COT, ZS_COT (sem_map.py:224)
- `**model_kwargs` — Truyen them cho model (sem_map.py:227)

### Output:
- DataFrame goc + column `suffix` (default `_map`) chua LLM output strings (sem_map.py:272-273)
- **return_explanations=True**: Them column `explanation_map` (sem_map.py:274-275)
- **return_raw_outputs=True**: Them column `raw_output_map` (sem_map.py:276-277)

## 7. Edge Cases

1. **Column khong ton tai**: Raise `ValueError` (sem_map.py:238-239)
2. **LM chua configure**: Raise `ValueError` (sem_map.py:229-232)
3. **Examples khong co "Answer" column**: Assert error (sem_map.py:249)
4. **COT/ZS_COT voi examples**: Tu dong set `return_explanations=True` va lay "Reasoning" column (sem_map.py:253-255) — co the raise error neu "Reasoning" column khong ton tai
5. **Custom postprocessor**: Phai co signature `(list[str], LM, bool) -> SemanticMapPostprocessOutput` (sem_map.py:19)

## 8. Code Examples

```python
# Basic mapping
df.sem_map("Label the sentiment of {text} as positive/negative")

# Voi custom system prompt
df.sem_map(
    "Classify the {document}",
    system_prompt="You are an expert document classifier."
)

# Voi few-shot examples
examples = pd.DataFrame({
    "text": ["Great product!", "Terrible service"],
    "Answer": ["positive", "negative"]
})
df.sem_map("Classify sentiment of {text}", examples=examples)

# Voi ZS-CoT
df.sem_map(
    "What is the main topic of {text}?",
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True
)

# Custom suffix
df.sem_map("Translate {text} to French", suffix="_french")
```

## 9. Assessment

### Diem manh:
- **Flexibility**: Custom system_prompt, custom postprocessor, **model_kwargs — rat linh hoat
- **Clean API**: Don gian, truc quan, dung nhu SQL SELECT + transform
- **Few-shot support**: Ho tro day du voi CoT reasoning examples

### Diem yeu:
- **Khong co cascade**: Khong nhu sem_filter, sem_map khong ho tro cascade optimization
- **No type validation**: Output la string, khong co validation output format
- **COT examples bug potential**: Khi `strategy=COT` ma examples khong co "Reasoning" column, se raise KeyError (sem_map.py:255) — khong co error message ro rang
- **Suffix collision**: Khong kiem tra xem suffix da ton tai trong DataFrame chua (sem_map.py:273), co the overwrite column
