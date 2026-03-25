# B3 — Langex Parsing

> **Phân tích** cách LOTUS parse Language Expressions (langex) để trích xuất column names và format instructions.

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
- `(?<!\{)` — Negative lookbehind: không có `{` trước
- `\{` — Match literal `{`
- `(?!\{)` — Negative lookahead: không có `{` sau
- `(.*?)` — Capture group: nội dung bên trong (non-greedy)
- `(?<!\})` — Negative lookbehind: không có `}` trước
- `\}` — Match literal `}`
- `(?!\})` — Negative lookahead: không có `}` sau

**Mục đích**: Match `{col}` nhưng KHÔNG match `{{escaped}}`.

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

**Mục đích**: Thay thế `{col}` bằng `Col` (capitalized) trong instruction string.

**Ví dụ**:
```python
nle2str("The {text} mentions {topic}", ["text", "topic"])
# → "The Text mentions Topic"
```

## 2. Parse Flow trong Operators

### Bước 1: parse_cols — Trích xuất column names
```python
col_li = lotus.nl_expression.parse_cols(user_instruction)
# Input: "The {text} has positive sentiment about {product}"
# Output: ["text", "product"]
```

### Bước 2: Column validation
```python
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

### Bước 3: df2multimodal_info — Serialize data
```python
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
# Trích xuất data từ các columns được reference
```

### Bước 4: nle2str — Format instruction
```python
formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)
# "The {text} has positive sentiment about {product}"
# → "The Text has positive sentiment about Product"
```

### Bước 5: Build prompt
```python
prompt = task_instructions.filter_formatter(
    model, multimodal_data[i], formatted_usr_instr, ...
)
```

## 3. Data Serialization

### df2text (task_instructions.py:325-361)

Format mỗi row thành string:

**DEFAULT format** (task_instructions.py:329):
```python
f"[{cols[i].capitalize()}]: «{x[cols[i]]}»\n"
```
Ví dụ: `"[Text]: «Great product!»\n[Rating]: «5»\n"`

**JSON format** (task_instructions.py:346):
```python
projected_df.to_json(orient="records", lines=True)
```
Ví dụ: `'{"text":"Great product!","rating":5}'`

**XML format** (task_instructions.py:347-359):
```python
projected_df.to_xml(...)
```
Ví dụ: `"<row><text>Great product!</text><rating>5</rating></row>"`

Format được cấu hình qua `lotus.settings.serialization_format` (task_instructions.py:343-358).

### df2multimodal_info (task_instructions.py:364-379)

Tách columns thành text và image, return list of dicts:
```python
[
    {
        "text": "[Text]: «Great product!»\n",
        "image": {"Photo": "data:image/png;base64,..."}
    },
    ...
]
```

## 4. Lưu Ý Quan Trọng

### Escaped Braces
- `{col}` → được parse thành column reference
- `{{escaped}}` → KHÔNG được parse, giữ nguyên
- Regex sử dụng lookbehind/lookahead để phân biệt (nl_expression.py:6)

### Column Name Restrictions
- Column names không thể chứa `{` hoặc `}` (sẽ confuse regex)
- Column names phân biệt hoa/thường: `{Text}` != `{text}`
- Khoảng trắng trong column name: `{my column}` được phép — regex match `.*?` non-greedy

### nle2str Side Effect
- `nle2str` capitalize column names trong output instruction
- Ví dụ: `{text}` → `Text`, `{user_name}` → `User_name`
- Điều này là ý định thiết kế: trong prompt, column names được viết hoa để rõ ràng

### parse_cols Raises khi không có columns
- Nếu instruction không có `{col}` nào: Raise `ValueError` (nl_expression.py:10-13)
- **Ngoại lệ**: sem_agg với `all_cols=True` không gọi parse_cols (sem_agg.py:370-371)
- **Ngoại lệ**: sem_extract không dùng parse_cols — dùng `input_cols` list trực tiếp

### Variable Shadowing Bug
- `nle2str` dùng `dict` làm variable name (nl_expression.py:18) — shadow built-in `dict`
- Không gây lỗi nhưng là bad practice
