# Prompt Template: sem_extract

## Formatter Function

`extract_formatter` tai `task_instructions.py:257`:

```python
def extract_formatter(
    model: lotus.models.LM,
    multimodal_data: dict[str, Any],
    output_cols: dict[str, str | None],
    extract_quotes: bool = True,
    strategy: ReasoningStrategy | None = None,
) -> list[dict[str, str]]:
```

## Verbatim System Prompt

### Voi quotes (extract_quotes=True) - task_instructions.py:293-299

```python
sys_instruction = (
    "The user will provide the columns that need to be extracted and some relevant context.\n"
    f"Your job is to extract these columns and provide only a concise value for each field "
    f"and the corresponding full quote for each field in the '{', '.join(quote_fields)}' fields.\n"
    f"Here is a description of each field: {output_cols_with_desc}\n"
    f"The response should be valid JSON format with the following fields: {fields_str}.\n"
)
```

Vi du voi `output_cols={"sentiment": "positive/negative", "topic": None}`:

```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field and the corresponding full quote for each field in the 'sentiment_quote, topic_quote' fields.
Here is a description of each field: {'sentiment': 'positive/negative', 'topic': 'topic'}
The response should be valid JSON format with the following fields: sentiment, topic, sentiment_quote, topic_quote.
```

### Khong co quotes (extract_quotes=False) - task_instructions.py:301-306

```python
sys_instruction = (
    "The user will provide the columns that need to be extracted and some relevant context.\n"
    f"Your job is to extract these columns and provide only a concise value for each field.\n"
    f"Here is a description of each field: {output_cols_with_desc}\n"
    f"The response should be valid JSON format with the following fields: {fields_str}.\n"
)
```

### Voi CoT (strategy=COT hoac ZS_COT) - task_instructions.py:277-290

Them CoT instructions vao system prompt (task_instructions.py:309-310):
```python
if cot_instruction:
    sys_instruction += "\n" + cot_instruction
```

CoT instruction:
```
Let's think step by step. Use the following format to provide your answer:
        Reasoning:
<Your reasoning here. Think through each extraction step by step.>

Answer: <Your answer here. Provide the JSON response with fields: sentiment, topic>

```

## Output Column Processing

### output_cols_with_desc (task_instructions.py:266):
```python
output_cols_with_desc = {col: col if desc is None else desc for col, desc in output_cols.items()}
```

Neu description la `None`, dung column name lam description.

### Quote fields (task_instructions.py:271):
```python
quote_fields = [f"{col}_quote" for col in output_col_names]
```

Moi output column co them 1 `_quote` field tuong ung.

## Message Structure

```python
messages = [
    {"role": "system", "content": sys_instruction},
    user_message_formatter(multimodal_data),  # Context only, no instruction tag
]
```

Luu y: `user_message_formatter` duoc goi KHONG co `user_instruction_with_tag` (task_instructions.py:314). Chi co context, khong co "Instruction:" prefix.

### DeepSeek ZS_COT (task_instructions.py:317-319):
Them them 1 user message voi deepseek instructions.

## JSON Response Format

Khi khong dung CoT, `sem_extract` yeu cau JSON response (sem_extract.py:92):
```python
lm_output = model(inputs, response_format={"type": "json_object"}, ...)
```

Khi dung CoT, khong su dung `response_format` de cho phep reasoning text (sem_extract.py:89-90):
```python
if strategy in [ReasoningStrategy.COT, ReasoningStrategy.ZS_COT]:
    lm_output = model(inputs, progress_bar_desc=progress_bar_desc)
```

## Postprocessing

`extract_postprocess` tai `postprocessors.py:149`:

```python
def extract_postprocess(llm_answers, model, cot_reasoning=False):
    if cot_reasoning:
        postprocessor = get_cot_postprocessor(model, for_extract=True)
        extract_data, explanations = postprocessor(llm_answers)
    else:
        extract_data = []
        for llm_answer in llm_answers:
            try:
                output = json.loads(llm_answer)
            except json.JSONDecodeError:
                output = {}
            output = {key: str(value) for key, value in output.items()}
            extract_data.append(output)
```

1. Parse JSON tu LLM output
2. Convert tat ca values sang string: `{key: str(value) for key, value in output.items()}` (postprocessors.py:176)
3. Fallback: empty dict `{}` khi JSON parse that bai

## Vi du output

Input: `{"text": "Great product! 5 stars. Fast shipping."}`
Output cols: `{"sentiment": "positive/negative", "rating": "1-5 scale"}`

Expected LLM response:
```json
{"sentiment": "positive", "rating": "5", "sentiment_quote": "Great product! 5 stars.", "rating_quote": "5 stars"}
```

## Phan tich

1. **Structured output**: `sem_extract` la operator duy nhat yeu cau JSON output format (qua `response_format={"type": "json_object"}`).

2. **Quote extraction**: Tinh nang doc dao - trich xuat ca gia tri va cau quote goc tu van ban. Giup verify ket qua extraction.

3. **Khong co few-shot**: `extract_formatter` khong ho tro examples. Chi co system prompt + context.

4. **All values stringify**: `str(value)` convert moi gia tri (postprocessors.py:176), co the mat type information (vi du number -> string).
