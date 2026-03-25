# E4 - Predicate on Text

## Tổng quan

LOTUS gửi toàn bộ langex như một "claim" cho LLM để đánh giá True/False. Không có structured predicate extraction - mọi thứ đều đi qua LLM reasoning.

---

## 1. filter_formatter - Tạo Claim Prompt

**Location**: `task_instructions.py:87-157`

```python
def filter_formatter(model, multimodal_data, user_instruction, ...):
    sys_instruction = """The user will provide a claim and some relevant context.
    Your job is to determine whether the claim is true for the given context."""  # :99-101

    # User message:
    messages.append(user_message_formatter(
        multimodal_data, f"Claim: {user_instruction}"   # :156
    ))
```

### Prompt structure
```
System: The user will provide a claim and some relevant context.
        Your job is to determine whether the claim is true for the given context.
        Answer: <Your answer here. The answer should be either True or False>

User: Context:
      [Description]: «Great wireless headphones»
      [Price]: «29.99»

      Claim: the Description matches wireless headphones and Price < 100
```

---

## 2. Không có Structured Predicate Extraction

Khi user viết:
```python
df.sem_filter("the {description} matches wireless and {price} < 100")
```

LOTUS **KHÔNG**:
- Parse "price < 100" thành pandas condition
- Tách structured predicates từ unstructured predicates
- Chạy structured predicates trước để pre-filter

Toàn bộ string được gửi cho LLM. LLM phải:
1. Hiểu "Description matches wireless" (semantic matching)
2. Hiểu "Price < 100" (numeric comparison)
3. Combine (AND logic)
4. Trả lời True/False

---

## 3. Complex Predicates - Implicit LLM Reasoning

LOTUS handle AND/OR/NOT qua LLM reasoning, không parse logic:

### AND
```python
df.sem_filter("the {description} is positive AND {rating} > 3")
```
LLM phải evaluate cả hai conditions và combine.

### OR
```python
df.sem_filter("the {description} mentions AI OR {category} is 'technology'")
```
LLM phải evaluate ít nhất một condition.

### NOT
```python
df.sem_filter("the {review} does NOT contain complaints")
```
LLM phải negate the condition.

### Nested
```python
df.sem_filter("({description} is about AI AND {price} < 50) OR {featured} is true")
```
LLM phải handle nested logic - accuracy depends on LLM capability.

---

## 4. User Instruction Template

Langex processing flow:

1. `parse_cols(user_instruction)` → extract column names (`nl_expression.py:4-14`)
2. `nle2str(user_instruction, col_li)` → replace `{col}` with `Col.capitalize()` (`nl_expression.py:17-21`)

Example:
```python
# Input: "the {description} matches and {price} < 100"
# After nle2str: "the Description matches and Price < 100"
```

Capitalized column names appear in both Context (as labels) and Claim (in instruction text).

---

## 5. Post-processing

`filter_postprocess` (`postprocessors.py:182-218`):

```python
def process_outputs(answer):
    if "True" in answer:     # :205 - simple string search
        return True
    elif "False" in answer:  # :207
        return False
    else:
        return default       # :210
```

- Simple "True"/"False" string matching
- Default fallback (configurable, default `True`)
- No confidence score returned to user (logprobs only used internally for cascade)

---

## 6. Implications

**Accuracy concerns**:
- LLM may fail at numeric comparisons ("is 99.99 < 100?")
- Complex boolean logic may confuse LLM
- No guarantee of consistency across similar rows

**Cost concerns**:
- Every row requires LLM call, even for simple structured predicates
- No pre-filtering optimization

**Flexibility benefits**:
- Can handle any natural language predicate
- No need to learn query syntax
- Fuzzy, contextual evaluation
