# B3 — Langex Parsing

> **Phan tich** cach LOTUS parse Language Expressions (langex) de trich xuat column names va format instructions.

## 1. Core Functions

### `parse_cols` — nl_expression.py:4-14

```python
def parse_cols(text: str) -> list[str]:
    pattern = r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"
    matches = re.findall(pattern, text)

    if not matches:
        raise ValueError(
            "Language expression contains no parameterized columns. "
            "Please specify the name of the relevant data column(s) "
            "in brackets {} within your language expression."
        )
    return matches
```

**Regex breakdown**:
- `(?<!\{)` — Negative lookbehind: khong co `{` truoc
- `\{` — Match literal `{`
- `(?!\{)` — Negative lookahead: khong co `{` sau
- `(.*?)` — Capture group: noi dung ben trong (non-greedy)
- `(?<!\})` — Negative lookbehind: khong co `}` truoc
- `\}` — Match literal `}`
- `(?!\})` — Negative lookahead: khong co `}` sau

**Muc dich**: Match `{col}` nhung KHONG match `{{escaped}}`.

**Test case** (nl_expression.py:25-29):
```python
text = "This is a {test} string with {variable} and {{escaped_variable}}."
assert parse_cols(text) == ["test", "variable"]
```

### `nle2str` — nl_expression.py:17-21

```python
def nle2str(nle: str, cols: list[str]) -> str:
    dict = {}
    for col in cols:
        dict[col] = f"{col.capitalize()}"
    return nle.format(**dict)
```

**Muc dich**: Thay the `{col}` bang `Col` (capitalized) trong instruction string.

**Vi du**:
```python
nle2str("The {text} mentions {topic}", ["text", "topic"])
# → "The Text mentions Topic"
```

## 2. Parse Flow trong Operators

### Buoc 1: parse_cols — Trich xuat column names
```python
col_li = lotus.nl_expression.parse_cols(user_instruction)
# Input: "The {text} has positive sentiment about {product}"
# Output: ["text", "product"]
```

### Buoc 2: Column validation
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

### Buoc 3: df2multimodal_info — Serialize data
```python
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
# Trich xuat data tu cac columns duoc reference
```

### Buoc 4: nle2str — Format instruction
```python
formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)
# "The {text} has positive sentiment about {product}"
# → "The Text has positive sentiment about Product"
```

### Buoc 5: Build prompt
```python
prompt = task_instructions.filter_formatter(
    model, multimodal_data[i], formatted_usr_instr, ...
)
```

## 3. Data Serialization

### df2text (task_instructions.py:325-361)

Format moi row thanh string:

**DEFAULT format** (task_instructions.py:329):
```python
f"[{cols[i].capitalize()}]: «{x[cols[i]]}»\n"
```
Vi du: `"[Text]: «Great product!»\n[Rating]: «5»\n"`

**JSON format** (task_instructions.py:346):
```python
projected_df.to_json(orient="records", lines=True)
```
Vi du: `'{"text":"Great product!","rating":5}'`

**XML format** (task_instructions.py:347-359):
```python
projected_df.to_xml(...)
```
Vi du: `"<row><text>Great product!</text><rating>5</rating></row>"`

Format duoc cau hinh qua `lotus.settings.serialization_format` (task_instructions.py:343-358).

### df2multimodal_info (task_instructions.py:364-379)

Tach columns thanh text va image, return list of dicts:
```python
[
    {
        "text": "[Text]: «Great product!»\n",
        "image": {"Photo": "data:image/png;base64,..."}
    },
    ...
]
```

## 4. Luu Y Quan Trong

### Escaped Braces
- `{col}` → duoc parse thanh column reference
- `{{escaped}}` → KHONG duoc parse, giu nguyen
- Regex su dung lookbehind/lookahead de phan biet (nl_expression.py:6)

### Column Name Restrictions
- Column names khong the chua `{` hoac `}` (se confuse regex)
- Column names phan biet hoa/thuong: `{Text}` != `{text}`
- Khoang trang trong column name: `{my column}` duoc phep — regex match `.*?` non-greedy

### nle2str Side Effect
- `nle2str` capitalize column names trong output instruction
- Vi du: `{text}` → `Text`, `{user_name}` → `User_name`
- Dieu nay la y dinh thiet ke: trong prompt, column names duoc viet hoa de ro rang

### parse_cols Raises khi khong co columns
- Neu instruction khong co `{col}` nao: Raise `ValueError` (nl_expression.py:10-13)
- **Ngoai le**: sem_agg voi `all_cols=True` khong goi parse_cols (sem_agg.py:370-371)
- **Ngoai le**: sem_extract khong dung parse_cols — dung `input_cols` list truc tiep

### Variable Shadowing Bug
- `nle2str` dung `dict` lam variable name (nl_expression.py:18) — shadow built-in `dict`
- Khong gay loi nhung la bad practice
