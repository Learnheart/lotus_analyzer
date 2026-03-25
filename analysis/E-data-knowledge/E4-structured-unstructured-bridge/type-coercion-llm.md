# E4 - Type Coercion from LLM

## Tổng quan

Tất cả LLM outputs ban đầu là strings. Mỗi operator có cách parse riêng, nhưng KHÔNG có automatic type conversion cho numeric, datetime, hay complex types.

---

## 1. All Outputs Start as Strings

LLM trả về text responses. LOTUS extract answer từ text:

```python
# lm.py:485-494
def _get_top_choice(self, response):
    choice = response.choices[0]
    return choice.message.content  # Always a string
```

---

## 2. sem_extract: JSON → String Dict

**Location**: `postprocessors.py:149-179`

```python
# Step 1: JSON parsing
try:
    output = json.loads(llm_answer)            # :170-171
except json.JSONDecodeError:
    output = {}                                 # :173-174

# Step 2: ALL values → str
output = {key: str(value) for key, value in output.items()}  # :176
```

Ví dụ:
- LLM returns: `{"count": 5, "active": true, "rate": 3.14}`
- After postprocess: `{"count": "5", "active": "True", "rate": "3.14"}`

Mọi type information bị mất tại line 176.

---

## 3. sem_filter: String → Boolean

**Location**: `postprocessors.py:200-211`

```python
def process_outputs(answer):
    if answer is None:
        return default                   # :202-203
    if "True" in answer:                 # :205
        return True
    elif "False" in answer:              # :207
        return False
    else:
        return default                   # :210
```

- Simple `"True"/"False"` substring search
- `"Partially True"` → `True` (contains "True")
- `"Not True at all"` → `True` (contains "True") - potential issue
- Default fallback: configurable, default `True` (`sem_filter.py:28`)

---

## 4. sem_topk: String → Document Number

**Location**: `sem_topk.py:83-129`

```python
def parse_ans_binary(answer):
    # Try "Document X" format first
    matches = list(re.finditer(r"Document[\s*](\d+)", answer, re.IGNORECASE))  # :119

    # Fallback: any number
    if len(matches) == 0:
        matches = list(re.finditer(r"(\d+)", answer, re.IGNORECASE))           # :121

    ans = int(matches[-1].group(1)) - 1    # :122 - convert to 0-indexed
    if ans not in [0, 1]:                   # :123
        return True, cot_explanation        # :125 - default Document 1
    return ans == 0, cot_explanation        # :126
```

- Regex extraction: `Document\s*(\d+)` hoặc fallback `(\d+)`
- Last match used (in case LLM mentions multiple numbers)
- Default to Document 1 (True) if can't parse

---

## 5. sem_map: No Parsing

sem_map output là raw string từ LLM - không có parsing:

```python
# postprocessors.py:123-146
def map_postprocess(llm_answers, model, cot_reasoning=False):
    if cot_reasoning:
        outputs, explanations = postprocessor(llm_answers)
    else:
        outputs = llm_answers  # Raw strings
```

---

## 6. sem_agg: No Parsing

sem_agg cũng trả về raw string:
```python
# sem_agg.py:213
summaries = lm_output.outputs  # Raw strings from LLM
```

---

## 7. Không có Automatic Type Conversion

LOTUS không cung cấp:
- `int()` conversion cho numeric outputs
- `float()` conversion cho decimal outputs
- `datetime` parsing
- `bool()` conversion (ngoại trừ filter)
- List/array parsing
- Nested object parsing

User phải tự convert:
```python
df = df.sem_extract(["text"], {"price": "the price in dollars"})
df["price"] = df["price"].astype(float)  # Manual conversion
```

---

## 8. CoT Postprocessing

Khi sử dụng Chain-of-Thought, postprocessors tách reasoning từ answer:

### Standard CoT (`postprocessors.py:12-43`)
```python
answer_idx = llm_answer.find("Answer:")
reasoning = llm_answer[reasoning_idx:answer_idx]
answer = llm_answer[answer_idx + len("Answer:"):].strip()
```

### DeepSeek CoT (`postprocessors.py:46-93`)
```python
think_start = llm_answer.find("<think>")
think_end = llm_answer.find("</think>")
reasoning = llm_answer[think_start + len("<think>"):think_end]
answer = llm_answer[answer_start + len("Answer:"):].strip()
```

---

## 9. Kết luận

Type coercion trong LOTUS:
- **sem_extract**: JSON parse → all values → str (line 176)
- **sem_filter**: "True"/"False" substring match → bool
- **sem_topk**: regex Document number → int → bool
- **sem_map/sem_agg**: no parsing, raw strings
- **No automatic numeric/datetime conversion**
