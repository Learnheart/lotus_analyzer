# Prompt Template: sem_map

## Formatter Function

`map_formatter` tai `task_instructions.py:213`:

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

User co the truyen `system_prompt` parameter de override hoan toan system instruction.

## 3 Sub-formatters

### 1. map_formatter (default) - task_instructions.py:213

Khi khong co CoT va khong co cot_reasoning:

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

Khi `cot_reasoning` duoc cung cap:

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

Them deepseek-specific CoT instructions vao cuoi user message.

## Full Message Structure (default, voi 1 example)

```python
messages = [
    {"role": "system", "content": "The user will provide an instruction and some relevant context.\nYour job is to answer the user's instruction given the context."},
    {"role": "user", "content": "Context:\n[Title]: <<Example Doc>>\n\nInstruction: Summarize {Title}"},
    {"role": "assistant", "content": "This is an example summary."},
    {"role": "user", "content": "Context:\n[Title]: <<Actual Doc>>\n\nInstruction: Summarize {Title}"},
]
```

## Postprocessing

`map_postprocess` tai `postprocessors.py:123`:

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

- **Khong co CoT**: Output la raw LLM answer, khong postprocess
- **Voi CoT**: Tach reasoning va answer qua `cot_postprocessor` (postprocessors.py:12)
- **DeepSeek**: Dung `deepseek_cot_postprocessor` (postprocessors.py:46) de parse `<think>` tags

## Phan tich

1. **Flexible output**: Khac voi `sem_filter` (chi True/False), `sem_map` cho phep LLM tra ve bat ky text nao. Khong co strict output format.

2. **Custom system_prompt**: User co the override system prompt de tuy chinh hanh vi LLM, vi du: su dung cho `llm_as_judge` voi system prompt "You are an intelligent, rigorous, and fair evaluator." (llm_as_judge.py:70-73).

3. **Postprocessor pluggable**: `sem_map` (sem_map.py:219) va `sem_extract` (sem_extract.py:207) cho phep truyen custom postprocessor function.

4. **Luu y loi typo**: Tai task_instructions.py:250 co `user_intructions` (thieu 's' -> `instructions`). Khong anh huong functionality nhung cho thay code co the chua bugs nho.
