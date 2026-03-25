# E5 - Hallucination Mitigation

## Tổng quan

LOTUS áp dụng một số strategies để giảm hallucination, nhưng không có comprehensive anti-hallucination framework. Chủ yếu dựa vào: temperature 0, context grounding, và source attribution.

---

## 1. Temperature 0.0 Default

**Location**: `lm.py:69-70`

```python
def __init__(
    self,
    model="gpt-4o-mini",
    temperature=0.0,       # :69 - deterministic output
    ...
):
```

Temperature 0 → greedy decoding → most likely token at each step. Giảm randomness → giảm creative hallucination.

---

## 2. Context Grounding

Mọi prompt đều include context từ actual data:

### user_message_formatter (`task_instructions.py:68-84`)
```python
return {
    "role": "user",
    "content": f"Context:\n{text}\n\n{user_instruction_with_tag}",  # :76
}
```

LLM luôn nhận context trước instruction → grounded trong actual data.

### Filter grounding
```
Context:
[Description]: «Actual product description»
[Price]: «29.99»

Claim: the Description mentions wireless headphones
```

LLM phải base answer trên provided context, không sáng tạo.

---

## 3. Extract Quotes - Source Attribution

**Location**: `task_instructions.py:270-272, 292-298`

```python
if extract_quotes:
    quote_fields = [f"{col}_quote" for col in output_col_names]  # :271
```

System prompt khi extract_quotes=True (`task_instructions.py:293-298`):
```
Your job is to extract these columns and provide only a concise value for each field
and the corresponding full quote for each field in the '{quote_fields}' fields.
```

Yêu cầu LLM cung cấp exact quote từ source text cho mỗi extracted value → traceable, verifiable.

---

## 4. filter_postprocess Default Fallback

**Location**: `postprocessors.py:200-211`

```python
def process_outputs(answer):
    if answer is None:
        lotus.logger.info(f"\t Failed to parse {answer}: defaulting to {default}")  # :202
        return default                                                                # :203
    if "True" in answer:
        return True
    elif "False" in answer:
        return False
    else:
        lotus.logger.info(f"\t Failed to parse {answer}: defaulting to {default}")  # :210
        return default                                                                # :211
```

Default fallback (`default=True` by default) khi LLM output không parseable. Đây là safety net - tốt hơn crash, nhưng có thể introduce false positives.

---

## 5. Không có Explicit Self-consistency

LOTUS **KHÔNG** implement:
- Self-consistency (multiple sampling + majority vote)
- Verification chain (generate → verify → correct)
- Multi-model consensus
- Fact-checking against external sources
- Confidence-based filtering (logprobs available nhưng không exposed to user)

---

## 6. Không có Majority Voting

Ngoại trừ `llm_as_judge` với `n_trials > 1` (`evals/llm_as_judge.py:22`), không có operator nào chạy multiple times và aggregate results.

Và ngay cả `llm_as_judge`, output là list of individual trial results - không có automatic majority vote:
```python
# llm_as_judge.py:106-114
outputs = []
for sem_map_output in sem_map_outputs:
    outputs.append(sem_map_output)  # Each trial separate
```

---

## 7. Structural Constraints

Một số operators có structural constraints giảm hallucination:

- **sem_filter**: Output constrained to True/False
- **sem_extract**: JSON response format enforced (`sem_extract.py:92`)
  ```python
  lm_output = model(inputs, response_format={"type": "json_object"}, ...)
  ```
- **sem_topk**: Output constrained to "Document 1" or "Document 2"

Nhưng content trong answer vẫn có thể hallucinate.

---

## 8. Kết luận

Hallucination mitigation trong LOTUS:
- **Có**: Temperature 0, context grounding, extract_quotes, structural constraints
- **Không có**: Self-consistency, majority voting, verification, fact-checking
- **Partial**: Default fallback cho unparseable outputs
- **Gap**: Logprobs available nhưng không exposed cho user-level confidence filtering
