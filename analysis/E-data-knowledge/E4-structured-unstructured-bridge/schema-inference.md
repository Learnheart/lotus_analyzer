# E4 - Schema Inference

## Tổng quan

LOTUS **KHÔNG** có automatic schema inference. Schema cho extraction luôn do user define qua `output_cols` dictionary.

---

## 1. User-defined Schema

`sem_extract` yêu cầu user cung cấp `output_cols`:

```python
df.sem_extract(
    input_cols=["text"],
    output_cols={
        "name": "person name",           # field name → description
        "email": "email address",
        "sentiment": "positive/negative/neutral"
    }
)
```

---

## 2. Schema trong Prompt

**Location**: `task_instructions.py:264-266`

```python
output_col_names = list(output_cols.keys())                       # :264
output_cols_with_desc = {
    col: col if desc is None else desc                            # :266
    for col, desc in output_cols.items()
}
```

Schema information được nhúng vào system prompt:
- Field names: liệt kê trong `fields_str` (`task_instructions.py:274`)
- Field descriptions: `output_cols_with_desc` dict (`task_instructions.py:297, 304`)

```
Here is a description of each field: {'name': 'person name', 'email': 'email address'}
The response should be valid JSON format with the following fields: name, email, name_quote, email_quote.
```

---

## 3. Không có Auto-inference

Không tìm thấy trong codebase:
- Automatic field detection từ text
- Schema learning từ examples
- Dynamic schema expansion
- Field type inference (mọi field đều là string output)

---

## 4. Description Optional

Nếu description là `None`, dùng field name làm description (`task_instructions.py:266`):

```python
# User provides:
output_cols = {"name": None, "email": None}

# Engine uses:
output_cols_with_desc = {"name": "name", "email": "email"}
```

---

## 5. Output Always String-typed

Bất kể field description nói gì, output luôn là strings:

```python
output = {key: str(value) for key, value in output.items()}  # postprocessors.py:176
```

Schema chỉ guide LLM extraction, không enforce type constraints.

---

## 6. Kết luận

Schema handling trong LOTUS:
- **Always user-specified**: không có auto-inference
- **Description-based**: guide LLM via prompt
- **String-only output**: no type enforcement
- **Flexible**: any field name/description combination
- **Simple**: no complex schema validation
