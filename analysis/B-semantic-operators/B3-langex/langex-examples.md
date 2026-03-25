# B3 — Langex Expression Examples

> **Ví dụ** về các langex expressions và cách chúng được xử lý qua pipeline.

## 1. Langex Syntax

### Basic syntax
```
{column_name}  → Reference to DataFrame column
{{escaped}}    → Literal braces, NOT parsed as column
```

### Regex (nl_expression.py:6):
```
r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"
```

## 2. Complete Processing Examples

### Example 1: sem_filter — Single Column

**Input:**
```python
df = pd.DataFrame({"text": ["Great product!", "Terrible service"]})
df.sem_filter("The {text} has positive sentiment")
```

**Step 1 — parse_cols** (nl_expression.py:4):
```python
parse_cols("The {text} has positive sentiment")
# → ["text"]
```

**Step 2 — Validate**: Check "text" in df.columns → OK (sem_filter.py:363-365)

**Step 3 — df2multimodal_info** (task_instructions.py:364):
```python
[
    {"text": "[Text]: «Great product!»\n", "image": {}},
    {"text": "[Text]: «Terrible service»\n", "image": {}}
]
```

**Step 4 — nle2str** (nl_expression.py:17):
```python
nle2str("The {text} has positive sentiment", ["text"])
# → "The Text has positive sentiment"
```

**Step 5 — filter_formatter** (task_instructions.py:87):
```
System: The user will provide a claim and some relevant context...
User: Context:
[Text]: «Great product!»

Claim: The Text has positive sentiment
```

**Step 6 — LLM output**: `"Answer: True"`

**Step 7 — filter_postprocess** (postprocessors.py:182): → `True`

---

### Example 2: sem_map — Multiple Columns

**Input:**
```python
df = pd.DataFrame({
    "title": ["AI Guide"],
    "author": ["John"]
})
df.sem_map("Describe the book {title} by {author}")
```

**Step 1 — parse_cols**:
```python
parse_cols("Describe the book {title} by {author}")
# → ["title", "author"]
```

**Step 3 — df2multimodal_info**:
```python
[{"text": "[Title]: «AI Guide»\n[Author]: «John»\n", "image": {}}]
```

**Step 4 — nle2str**:
```python
# → "Describe the book Title by Author"
```

**Step 5 — map_formatter** (task_instructions.py:213):
```
System: The user will provide an instruction and some relevant context...
User: Context:
[Title]: «AI Guide»
[Author]: «John»

Instruction: Describe the book Title by Author
```

---

### Example 3: sem_join — Cross-DataFrame Columns

**Input:**
```python
df1 = pd.DataFrame({"article": ["ML tutorial"]})
df2 = pd.DataFrame({"category": ["Computer Science"]})
df1.sem_join(df2, "the {article} belongs to the {category}")
```

**Step 1 — parse_cols**:
```python
parse_cols("the {article} belongs to the {category}")
# → ["article", "category"]
```

**Step 2 — Column resolution** (sem_join.py:710-727):
- "article" in df1.columns → `left_on = "article"`
- "category" in df2.columns → `right_on = "category"`

**Step 3 — merge_multimodal_info** (task_instructions.py:382):
```python
# Left: [{"text": "[Article]: «ML tutorial»\n", "image": {}}]
# Right: [{"text": "[Category]: «Computer Science»\n", "image": {}}]
# Merged: [{"text": "[Article]: «ML tutorial»\n\n[Category]: «Computer Science»\n", "image": {}}]
```

**Step 4 — filter_formatter** (dùng sem_filter nội bộ):
```
System: The user will provide a claim and some relevant context...
User: Context:
[Article]: «ML tutorial»
[Category]: «Computer Science»

Claim: the {article} belongs to the {category}
```

Lưu ý: sem_join truyền `join_instruction` trực tiếp cho sem_filter, KHÔNG dùng nle2str.

---

### Example 4: sem_join — Disambiguated Columns

**Input:**
```python
df1 = pd.DataFrame({"name": ["Alice"]})
df2 = pd.DataFrame({"name": ["Alice Smith"]})
df1.sem_join(df2, "the {name:left} is the same person as {name:right}")
```

**Step 1 — parse_cols**:
```python
parse_cols("the {name:left} is the same person as {name:right}")
# → ["name:left", "name:right"]
```

**Step 2 — Column resolution** (sem_join.py:700-708):
- "name:left" → `left_on = "name:left"`, `real_left_on = "name"`
- "name:right" → `right_on = "name:right"`, `real_right_on = "name"`

---

### Example 5: sem_agg — all_cols Mode

**Input:**
```python
df = pd.DataFrame({
    "journal": ["Happy day", "Sad day"],
    "date": ["Mon", "Tue"]
})
df.sem_agg("Summarize the key points", all_cols=True)
```

**Flow**: Skip parse_cols! (sem_agg.py:370-371)
```python
col_li = list(self._obj.columns)  # ["journal", "date"]
```

---

### Example 6: sem_extract — No Langex

**Input:**
```python
df.sem_extract(
    ["text"],
    {"sentiment": "positive/negative", "rating": "1-5"}
)
```

**Flow**: Không dùng parse_cols! (sem_extract.py:222)
```python
# input_cols = ["text"] — truyền trực tiếp
# output_cols = {"sentiment": "positive/negative", "rating": "1-5"}
```

---

### Example 7: Escaped Braces

**Input:**
```python
df.sem_filter("The {text} contains {{JSON}} data")
```

**Step 1 — parse_cols**:
```python
parse_cols("The {text} contains {{JSON}} data")
# → ["text"]  ({{JSON}} được bỏ qua)
```

**Step 4 — nle2str**:
```python
nle2str("The {text} contains {{JSON}} data", ["text"])
# → "The Text contains {JSON} data"
# Lưu ý: Python .format() chuyển {{}} thành {}
```

## 3. Error Cases

### Case 1: No columns in langex
```python
df.sem_filter("All documents are positive")
# → ValueError: "Language expression contains no parameterized columns..."
# (nl_expression.py:10-13)
```

### Case 2: Column not in DataFrame
```python
df = pd.DataFrame({"text": ["hello"]})
df.sem_filter("The {title} is positive")
# → ValueError: "Column title not found in DataFrame"
# (sem_filter.py:365)
```

### Case 3: Ambiguous join column
```python
df1 = pd.DataFrame({"name": ["A"]})
df2 = pd.DataFrame({"name": ["B"]})
df1.sem_join(df2, "the {name} matches")
# → ValueError: "Column found in both dataframes"
# (sem_join.py:717)
```

## 4. Summary

| Langex Pattern | parse_cols Output | nle2str Output |
|---|---|---|
| `"The {text} is good"` | `["text"]` | `"The Text is good"` |
| `"{title} by {author}"` | `["title", "author"]` | `"Title by Author"` |
| `"{name:left} = {name:right}"` | `["name:left", "name:right"]` | `"Name:left = Name:right"` |
| `"Has {{JSON}} in {text}"` | `["text"]` | `"Has {JSON} in Text"` |
| `"No columns here"` | `ValueError` | N/A |
