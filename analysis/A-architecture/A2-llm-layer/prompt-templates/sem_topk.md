# Prompt Template: sem_topk

## Formatter Function

`get_match_prompt_binary` tai `sem_topk.py:16`:

```python
def get_match_prompt_binary(
    doc1: dict[str, Any],
    doc2: dict[str, Any],
    user_instruction: str,
    model: lotus.models.LM,
    strategy: ReasoningStrategy | None = None,
) -> list[dict[str, Any]]:
```

## Verbatim System Prompt

### Default (no CoT) - sem_topk.py:59-66

```python
sys_prompt = (
    "Your job is to to select and return the most relevant document to the user's question.\n"
    "Carefully read the user's question and the two documents provided below.\n"
    'Respond only with the label of the document such as "Document NUMBER".\n'
    "NUMBER must be either 1 or 2, depending on which document is most relevant.\n"
    'You must pick a number and cannot say things like "None" or "Neither"'
)
```

Verbatim:
```
Your job is to to select and return the most relevant document to the user's question.
Carefully read the user's question and the two documents provided below.
Respond only with the label of the document such as "Document NUMBER".
NUMBER must be either 1 or 2, depending on which document is most relevant.
You must pick a number and cannot say things like "None" or "Neither"
```

Luu y: co typo "to to" (sem_topk.py:53).

### ZS_COT - sem_topk.py:52-58

```python
sys_prompt = (
    "Your job is to to select and return the most relevant document to the user's question.\n"
    "Carefully read the user's question and the two documents provided below.\n"
    'First give your reasoning. Then you MUST end your output with "Answer: Document 1 or Document 2"\n'
    'You must pick a number and cannot say things like "None" or "Neither"\n'
    'Remember to explicitly state "Answer:" at the end before your choice.'
)
```

### DeepSeek ZS_COT - sem_topk.py:72-76

Khi `strategy == ReasoningStrategy.ZS_COT and model.is_deepseek()`:

```python
deepseek_instructions = """Please think through your reasoning step by step, then provide your final answer.
        You must put your reasoning insdie the <think></think> tags, then provide your
        final answer after the </think> tag with the format: Answer: your answer."""
prompt += [{"type": "text", "text": f"\n{deepseek_instructions}"}]
```

Luu y: co typo "insdie" (sem_topk.py:74).

## User Message Format

```python
prompt = [{"type": "text", "text": f"Question: {user_instruction}\n"}]
for idx, doc in enumerate([doc1, doc2]):
    content_text, content_image_inputs = task_instructions.context_formatter(doc)
    prompt += [{"type": "text", "text": f"\nDocument {idx+1}:\n{content_text}"}, *content_image_inputs]
```

Vi du:
```
Question: Which tutorial is best for beginners?

Document 1:
[Title]: <<Machine Learning 101>>

Document 2:
[Title]: <<Advanced Deep Learning>>
```

## Full Message Structure

```python
messages = [
    {"role": "system", "content": sys_prompt},
    {"role": "user", "content": [
        {"type": "text", "text": "Question: Which is best for beginners?\n"},
        {"type": "text", "text": "\nDocument 1:\n[Title]: <<ML 101>>\n"},
        {"type": "text", "text": "\nDocument 2:\n[Title]: <<Advanced DL>>\n"},
    ]}
]
```

Luu y: `content` luon la list of content parts (multimodal format), ke ca khi khong co images (sem_topk.py:67-78).

## Response Parsing

`parse_ans_binary` tai `sem_topk.py:83`:

```python
def parse_ans_binary(answer: str) -> tuple[bool, str]:
    # Handle <think> tags (DeepSeek)
    think_start = answer.find("<think>")
    think_end = answer.find("</think>")
    if think_start != -1 and think_end != -1:
        cot_explanation = answer[think_start + len("<think>"):think_end].strip()
        answer = answer[think_end + len("</think>"):].strip()
    else:
        # Handle "Answer:" prefix
        answer_idx = answer.lower().find("answer:")
        if answer_idx != -1:
            cot_explanation = answer[:answer_idx].strip()
            answer = answer[answer_idx:].strip()

    # Find document number
    matches = list(re.finditer(r"Document[\s*](\d+)", answer, re.IGNORECASE))
    if len(matches) == 0:
        matches = list(re.finditer(r"(\d+)", answer, re.IGNORECASE))
    ans = int(matches[-1].group(1)) - 1

    if ans not in [0, 1]:
        return True, cot_explanation  # Default: Document 1
    return ans == 0, cot_explanation
```

- Tim "Document 1" hoac "Document 2" trong response
- Fallback: tim bat ky so nao
- Default: `True` (Document 1 thang) khi khong parse duoc

## Sorting Algorithms

`sem_topk` su dung binary comparison cho nhieu thuat toan:

### 1. llm_quicksort (sem_topk.py:347)
- Modified quicksort chi sort top-K
- `compare_batch_binary()` (sem_topk.py:132) so sanh batch documents voi pivot
- O(n log n) expected, nhung chi sort deep cho top-K

### 2. llm_heapsort (sem_topk.py:560)
- Su dung `heapq.nsmallest(K, heap)` (sem_topk.py:613)
- `HeapDoc.__lt__` (sem_topk.py:526) goi LLM de so sanh
- O(n log K)

### 3. llm_naive_sort (sem_topk.py:276)
- All-pairs comparison: O(n^2)
- Moi cap duoc so sanh, doc co nhieu "wins" xep cao hon

### 4. Cascade variant (sem_topk.py:176)
- `compare_batch_binary_cascade()`: dung helper_lm truoc, chi gui low-confidence pairs den oracle

## Phan tich

1. **Binary comparison**: Thay vi rank tat ca documents cung luc, `sem_topk` chi so sanh 2 documents moi lan. Day la pairwise comparison paradigm.

2. **Position bias**: Document 1 luon duoc liet ke truoc Document 2. Khong co randomization cua thu tu documents. Khi parse that bai, default chon Document 1 (`return True`), tao subtle bias.

3. **Multimodal format luon duoc su dung**: User content luon la list of parts, ke ca text-only (sem_topk.py:67-78). Dieu nay khac voi `filter_formatter` va `map_formatter` (dung string khi khong co images).

4. **Khong co few-shot**: `get_match_prompt_binary` khong ho tro examples.
