# Prompt Template: sem_map

## Formatter Function

`map_formatter` tại `task_instructions.py:213`:

```python
def map_formatter(
    model: lotus.models.LM,
    multimodal_data: dict[str, Any],
    user_instruction: str,
    examples_multimodal_data: list[dict[str, Any]] | None = None,
    examples_answer: list[str] | None = None,
    cot_reasoning: list[str] | None = None,
    strategy: ReasoningStrategy | str | None = None,
    system_prompt: str | None = None,
) -> list[dict[str, str]]:
```

## Verbatim System Prompt

### Default - task_instructions.py:223-226

```python
sys_instruction = system_prompt or (
    "The user will provide an instruction and some relevant context.\n"
    "Your job is to answer the user's instruction given the context."
)
```

Verbatim:
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.
```

### Custom system_prompt

User có thể truyền `system_prompt` parameter để override hoàn toàn system instruction.

## 3 Sub-formatters

### 1. map_formatter (default) - task_instructions.py:213

Khi không có CoT và không có cot_reasoning:

```python
messages = [
    {"role": "system", "content": sys_instruction},
]
# Few-shot examples
if examples_multimodal_data:
    for ex_df_txt, ex_ans in zip(examples_multimodal_data, examples_answers):
        messages.extend([
            user_message_formatter(ex_df_txt, f"Instruction: {user_instruction}"),
            {"role": "assistant", "content": str(ex_ans)},
        ])
# Actual input
messages.append(user_message_formatter(multimodal_data, f"Instruction: {user_instruction}"))
```

### 2. map_formatter_cot - task_instructions.py:160

Khi `cot_reasoning` được cung cấp:

System prompt (task_instructions.py:168-172):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.You must give your reasoning and then your final answer
```

Assistant examples format (task_instructions.py:186-188):
```
Reasoning:
{cot}

Answer: {ex_ans}
```

### 3. map_formatter_zs_cot - task_instructions.py:195

Khi `strategy == ReasoningStrategy.ZS_COT`:

System prompt (task_instructions.py:200-203):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.First give your reasoning. Then you MUST end your output with "Answer: your answer"
```

## User Message Format

```
Context:
[Title]: <<Machine Learning 101>>
[Author]: <<John>>

Instruction: Summarize the content of {Title}
```

Format: `"Context:\n{text}\n\nInstruction: {user_instruction}"`

## DeepSeek ZS_COT (task_instructions.py:249-251)

Khi `strategy == ReasoningStrategy.ZS_COT and model.is_deepseek()`:

```python
user_intructions = f"Instruction: {user_instruction}\n\n{deepseek_cot_formatter()}"
```

Thêm deepseek-specific CoT instructions vào cuối user message.

## Full Message Structure (default, với 1 example)

```python
messages = [
    {"role": "system", "content": "The user will provide an instruction and some relevant context.\nYour job is to answer the user's instruction given the context."},
    {"role": "user", "content": "Context:\n[Title]: <<Example Doc>>\n\nInstruction: Summarize {Title}"},
    {"role": "assistant", "content": "This is an example summary."},
    {"role": "user", "content": "Context:\n[Title]: <<Actual Doc>>\n\nInstruction: Summarize {Title}"},
]
```

## Postprocessing

`map_postprocess` tại `postprocessors.py:123`:

```python
def map_postprocess(llm_answers, model, cot_reasoning=False):
    if cot_reasoning:
        postprocessor = get_cot_postprocessor(model)
        outputs, explanations = postprocessor(llm_answers)
    else:
        outputs = llm_answers
        explanations = [None] * len(llm_answers)
    return SemanticMapPostprocessOutput(raw_outputs=llm_answers, outputs=outputs, explanations=explanations)
```

- **Không có CoT**: Output là raw LLM answer, không postprocess
- **Với CoT**: Tách reasoning và answer qua `cot_postprocessor` (postprocessors.py:12)
- **DeepSeek**: Dùng `deepseek_cot_postprocessor` (postprocessors.py:46) để parse `<think>` tags

## Phân tích

1. **Flexible output**: Khác với `sem_filter` (chỉ True/False), `sem_map` cho phép LLM trả về bất kỳ text nào. Không có strict output format.

2. **Custom system_prompt**: User có thể override system prompt để tùy chỉnh hành vi LLM, ví dụ: sử dụng cho `llm_as_judge` với system prompt "You are an intelligent, rigorous, and fair evaluator." (llm_as_judge.py:70-73).

3. **Postprocessor pluggable**: `sem_map` (sem_map.py:219) và `sem_extract` (sem_extract.py:207) cho phép truyền custom postprocessor function.

4. **Lưu ý lỗi typo**: Tại task_instructions.py:250 có `user_intructions` (thiếu 's' -> `instructions`). Không ảnh hưởng functionality nhưng cho thấy code có thể chứa bugs nhỏ.
