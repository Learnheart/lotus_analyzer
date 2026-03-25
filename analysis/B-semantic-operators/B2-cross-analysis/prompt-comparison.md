# B2 — Prompt Comparison Matrix

> **So sánh** các prompt templates giữa các LLM-based operators.

## Prompt Matrix

| Feature | sem_filter | sem_map | sem_join | sem_extract | sem_topk | sem_agg |
|---|---|---|---|---|---|---|
| **Formatter function** | `filter_formatter` (task_instructions.py:87) | `map_formatter` (task_instructions.py:213) | `filter_formatter` (dùng nội bộ sem_filter) | `extract_formatter` (task_instructions.py:257) | `get_match_prompt_binary` (sem_topk.py:16) | Inline templates (sem_agg.py:12-57) |
| **System instruction** | Yes (task_instructions.py:99-101) | Yes (task_instructions.py:223-226) | Giống sem_filter | Yes (task_instructions.py:293-306) | Yes (sem_topk.py:52-66) | No — chỉ user message |
| **User message tag** | `Claim: {instruction}` (task_instructions.py:156) | `Instruction: {instruction}` (task_instructions.py:253) | `Claim: {instruction}` | No tag — chỉ Context | `Question: {instruction}` (sem_topk.py:67) | `Instruction: {instruction}` (sem_agg.py:30) |
| **Output format** | True/False (task_instructions.py:97) | Free text | True/False (qua sem_filter) | JSON (task_instructions.py:298-305) | Document 1/2 (sem_topk.py:63) | Free text |
| **Few-shot support** | Yes (task_instructions.py:118-151) | Yes (task_instructions.py:239-246) | Yes (qua sem_filter) | No | No | No |
| **CoT support** | Yes — COT + ZS_COT | Yes — COT + ZS_COT | Yes (qua sem_filter) | Yes — COT + ZS_COT | Yes — ZS_COT only | No |
| **Custom system_prompt** | No | Yes (sem_map.py:218) | No | No | No | No |
| **response_format** | None | None | None | `{"type": "json_object"}` (sem_extract.py:92) | None | None |
| **DeepSeek special** | Yes (task_instructions.py:152-154) | Yes (task_instructions.py:249-251) | Yes (qua sem_filter) | Yes (task_instructions.py:317-319) | Yes (sem_topk.py:72-76) | No |

## System Instruction chi tiết

### sem_filter (task_instructions.py:99-101):
```
The user will provide a claim and some relevant context.
Your job is to determine whether the claim is true for the given context.
```
Thêm CoT hoặc non-CoT format instructions.

### sem_map (task_instructions.py:223-226):
```
The user will provide an instruction and some relevant context.
Your job is to answer the user's instruction given the context.
```
Cho phép custom system_prompt thay thế.

### sem_extract (task_instructions.py:301-305):
```
The user will provide the columns that need to be extracted and some relevant context.
Your job is to extract these columns and provide only a concise value for each field.
Here is a description of each field: {output_cols_with_desc}
The response should be valid JSON format with the following fields: {fields_str}.
```

### sem_topk (sem_topk.py:60-66):
```
Your job is to to select and return the most relevant document to the user's question.
Carefully read the user's question and the two documents provided below.
Respond only with the label of the document such as "Document NUMBER".
NUMBER must be either 1 or 2, depending on which document is most relevant.
You must pick a number and cannot say things like "None" or "Neither"
```

### sem_agg — leaf template (sem_agg.py:22-31):
```
Your job is to provide an answer to the user's instruction given the context below from multiple documents.
Remember that your job is to answer the user's instruction by combining all relevant information...
```

## Few-Shot Comparison

| Operator | Example format | Example DataFrame requirements |
|---|---|---|
| sem_filter | User: Context+Claim → Assistant: "Answer: True/False" | Columns matching langex + "Answer" (bool) + optional "Reasoning" |
| sem_map | User: Context+Instruction → Assistant: answer string | Columns matching langex + "Answer" (str) + optional "Reasoning" |
| sem_join | User: merged Context+Claim → Assistant: "Answer: True/False" | Columns from both DFs + "Answer" (bool) |
| sem_extract | Không hỗ trợ few-shot | — |
| sem_topk | Không hỗ trợ few-shot | — |
| sem_agg | Không hỗ trợ few-shot | — |

## CoT Strategy Comparison

| Strategy | sem_filter | sem_map | sem_extract | sem_topk |
|---|---|---|---|---|
| **None** | `Answer: True/False` | Free text | JSON | `Document 1/2` |
| **COT** | `Reasoning: ...\nAnswer: True/False` | `Reasoning: ...\nAnswer: ...` | `Reasoning: ...\nAnswer: {JSON}` | Không hỗ trợ |
| **ZS_COT** | `Reasoning: ...\nAnswer: True/False` | `Reasoning: ...\nAnswer: ...` | `Reasoning: ...\nAnswer: {JSON}` | `Reasoning + Answer: Document 1/2` |

### CoT format chi tiết (task_instructions.py:25-30):
```
Let's think step by step. Use the following format to provide your answer:
        Reasoning:
<Your reasoning here.>

Answer: <Your answer here.>
```

### DeepSeek CoT format (task_instructions.py:19-22):
```
Please think through your reasoning step by step, then provide your final answer.
You must put your reasoning inside the <think></think> tags, then provide your
final answer after the </think> tag with the format: Answer: your answer.
```

## Context Formatting

**Tất cả operators** dùng `context_formatter()` (task_instructions.py:40-65) để format multimodal data:
- Text format: `"[{col.capitalize()}]: {value}\n"` (task_instructions.py:329)
- Image format: `{"type": "image_url", ...}` (task_instructions.py:55-58)

**User message wrapper** `user_message_formatter()` (task_instructions.py:68-84):
- Text-only: `"Context:\n{text}\n\n{instruction_tag}"`
- Multimodal: Content array với text + image items

## Nhận xét

1. **sem_join dùng filter prompt**: Điều này hợp lý vì join = filter trên pairs, nhưng làm cho join không có custom prompt format
2. **sem_extract khác biệt**: Dùng input_cols thay vì langex, dùng JSON response_format — API khác hẳn các operator khác
3. **sem_topk unique prompt**: Prompt so sánh 2 documents — khác hẳn với các operators khác
4. **sem_agg không có system message**: Chỉ dùng user message với inline template — khác với tất cả operators khác
5. **DeepSeek handling**: 5/6 LLM operators có DeepSeek special handling — nhưng sem_agg không có
6. **Custom system_prompt**: Chỉ sem_map hỗ trợ — các operators khác đều hardcoded
