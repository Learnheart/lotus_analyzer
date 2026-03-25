# Prompt Template: sem_filter

## Formatter Function

`filter_formatter` tại `task_instructions.py:87`:

```python
def filter_formatter(
    model: lotus.models.LM,
    multimodal_data: dict[str, Any],
    user_instruction: str,
    examples_multimodal_data: list[dict[str, Any]] | None = None,
    examples_answer: list[bool] | None = None,
    cot_reasoning: list[str] | None = None,
    strategy: ReasoningStrategy | None = None,
    reasoning_instructions: str = "",
) -> list[dict[str, str]]:
```

## Verbatim System Prompt

### Default (no CoT) - task_instructions.py:99-112

System instruction (task_instructions.py:99-101):
```
The user will provide a claim and some relevant context.
    Your job is to determine whether the claim is true for the given context.

```

Khi `strategy` KHÔNG phải COT/ZS_COT, thêm (task_instructions.py:112):
```
Use the following format to provide your answer:
            Answer: <Your answer here. The answer should be either True or False>

```

### COT / ZS_COT - task_instructions.py:103-110

System instruction + CoT format (task_instructions.py:103-106):
```
The user will provide a claim and some relevant context.
    Your job is to determine whether the claim is true for the given context.
     Let's think step by step. Use the following format to provide your answer:
        Reasoning:
<Your reasoning here. >

Answer: <Your answer here. The answer should be either True or False>

```

### DeepSeek ZS_COT - task_instructions.py:152-154

Khi `strategy == ReasoningStrategy.ZS_COT and model.is_deepseek()`:

User instruction được append:
```
Please think through your reasoning step by step, then provide your final answer.
    You must put your reasoning inside the <think></think> tags, then provide your
    final answer after the </think> tag with the format: Answer: your answer.
```

## User Message Format

Được tạo bởi `user_message_formatter` (task_instructions.py:68):

```
Context:
[Title]: <<Machine Learning 101>>

Claim: the Title is about AI
```

Format: `"Context:\n{text}\n\nClaim: {user_instruction}"` (task_instructions.py:76, 156)

## Few-shot Examples

Khi `examples_multimodal_data` và `examples_answer` được cung cấp (task_instructions.py:118-151):

Mỗi example tạo 2 messages:
1. User message: context + claim (task_instructions.py:145)
2. Assistant message: answer (task_instructions.py:141-146)

### Với CoT reasoning:
```
Reasoning:
{cot_reasoning[idx]}

Answer: {ex_ans}
```

### Không có CoT:
```
Answer: {ex_ans}
```

## Full Message Structure

```python
messages = [
    {"role": "system", "content": sys_instruction},
    # Few-shot examples (nếu có):
    {"role": "user", "content": "Context:\n{example_text}\n\nClaim: {instruction}"},
    {"role": "assistant", "content": "Answer: True"},
    {"role": "user", "content": "Context:\n{example_text}\n\nClaim: {instruction}"},
    {"role": "assistant", "content": "Answer: False"},
    # Actual input:
    {"role": "user", "content": "Context:\n{actual_text}\n\nClaim: {instruction}"},
]
```

## Postprocessing

`filter_postprocess` tại `postprocessors.py:182`:

```python
def filter_postprocess(llm_answers, model, default=True):
    postprocessor = get_cot_postprocessor(model)
    outputs, explanations = postprocessor(llm_answers)

    def process_outputs(answer):
        if answer is None:
            return default
        if "True" in answer:
            return True
        elif "False" in answer:
            return False
        else:
            return default

    boolean_outputs = [process_outputs(answer) for answer in outputs]
    return SemanticFilterPostprocessOutput(...)
```

1. Gọi `cot_postprocessor` để tách reasoning và answer (postprocessors.py:12-43)
2. Tìm "True" hoặc "False" trong answer string (postprocessors.py:205-208)
3. Fallback: `default` parameter (thông thường `True`)

## Phân tích

1. **Claim-based framing**: Prompt frame bài toán như kiểm chứng claim (True/False), không phải classification. Điều này giúp LLM trả lời nhất quán hơn.

2. **answer_instructions** buộc LLM ghi rõ "True" hoặc "False" để dễ parse.

3. **Few-shot**: Examples được chèn thành user-assistant turn pairs, theo format của OpenAI chat API.

4. **Default value**: Nếu LLM output không parse được, `default=True` nghĩa là mặc định KEEP row (conservative filter). User có thể đổi thành `False` để mặc định REMOVE.
