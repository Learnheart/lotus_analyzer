# E1 - Structured Data Handling

## Cơ chế serialization

LOTUS serialize TẤT CẢ columns (cả structured lẫn unstructured) thành text để gửi cho LLM. Không có xử lý đặc biệt nào cho kiểu dữ liệu numeric.

---

## 1. df2text - Hàm serialization chính

**Location**: `task_instructions.py:325-361`

```python
def df2text(df: pd.DataFrame, cols: list[str]) -> list[str]:
```

### Format mặc định (DEFAULT)

Mỗi row được format theo pattern (`task_instructions.py:329`):
```
[Col_name]: «value»\n
```

Ví dụ với DataFrame `{price: 29.99, description: "Great product"}`:
```
[Price]: «29.99»
[Description]: «Great product»
```

Giá trị structured (int, float, datetime) chỉ đơn giản được `toString` - không có special handling (`task_instructions.py:329`).

### Format JSON

Khi `lotus.settings.serialization_format == SerializationFormat.JSON` (`task_instructions.py:345-346`):
```python
formatted_rows = projected_df.to_json(orient="records", lines=True).splitlines()
```
Output: `{"price": 29.99, "description": "Great product"}`

### Format XML

Khi `lotus.settings.serialization_format == SerializationFormat.XML` (`task_instructions.py:347-359`):
- Dùng `pandas.to_xml` với `pretty_print=False`
- Column names được clean bằng regex: `re.sub(r"[^\w]", "", column_name)` (`task_instructions.py:332`)
- Output: `<row><price>29.99</price><description>Great product</description></row>`

---

## 2. SerializationFormat enum

**Location**: `types.py:206-209`

```python
class SerializationFormat(Enum):
    JSON = "json"
    XML = "xml"
    DEFAULT = "default"
```

Ba format được hỗ trợ, configurable qua `lotus.settings.serialization_format`.

---

## 3. Structured data gửi cho LLM như text

Điểm quan trọng: structured values (int, float, boolean) **KHÔNG** được xử lý đặc biệt. Chúng đều được convert thành string và nhúng vào prompt:

- `task_instructions.py:329`: `f"[{cols[i].capitalize()}]: «{x[cols[i]]}»\n"` - dùng Python f-string, giá trị nào cũng được format giống nhau
- Không có type-aware formatting (ví dụ: không format số với thousand separators, không format date theo ISO)
- Không có metadata về data type gửi kèm trong prompt

---

## 4. Column filtering

Chỉ các columns được reference trong langex mới được serialize (`task_instructions.py:336`):
```python
cols = [col for col in cols if col in df.columns]
```

Nếu không có column nào match, trả về empty strings (`task_instructions.py:337-338`):
```python
if len(cols) == 0:
    return [""] * len(df)
```

---

## 5. Implications

- **Pro**: Đơn giản, LLM có thể hiểu mọi kiểu dữ liệu vì đều là text
- **Con**: Mất thông tin type - LLM phải tự suy luận "29.99" là số, không phải string
- **Con**: Không tối ưu cho numeric operations - LLM xử lý "price < 100" kém hơn SQL engine
- **Con**: Không có schema information trong prompt - LLM không biết column type
