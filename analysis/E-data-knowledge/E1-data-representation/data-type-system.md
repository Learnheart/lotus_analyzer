# E1 - Data Type System

## Tổng quan

LOTUS sử dụng pandas dtypes làm nền tảng, bổ sung thêm custom `ImageDtype` cho multimodal support. Hệ thống type cho LLM output đơn giản - hầu hết outputs là strings, booleans, hoặc dicts.

---

## 1. Pandas Dtypes + ImageDtype

### ImageDtype

**Location**: `dtype_extensions/image.py:12-35`

```python
class ImageDtype(ExtensionDtype):
    name = "image"
    type = Image.Image
    na_value = None
```

- Custom pandas ExtensionDtype cho image columns
- Backed bởi `ImageArray` (`dtype_extensions/image.py:37-305`) - ExtensionArray lưu trữ image references
- Hỗ trợ caching: `_cached_images` dict (`dtype_extensions/image.py:61`)
- Nhận dạng bởi `isinstance(df[col].dtype, ImageDtype)` (`task_instructions.py:369`)

### Standard Pandas Dtypes

Các column thông thường dùng pandas native dtypes:
- `int64`, `float64` cho numeric
- `object` cho text/string
- `datetime64` cho timestamps
- Không có custom dtype cho text hay structured data

---

## 2. LLM Output Types

### sem_map → strings

Output là raw string từ LLM. Khi có CoT reasoning, `cot_postprocessor` (`postprocessors.py:12-43`) tách reasoning và answer, nhưng answer vẫn là string.

### sem_filter → booleans

`filter_postprocess` (`postprocessors.py:182-218`):
```python
def process_outputs(answer):
    if "True" in answer:    # postprocessors.py:205
        return True
    elif "False" in answer: # postprocessors.py:207
        return False
    else:
        return default      # postprocessors.py:210
```

Simple string matching - tìm "True" hoặc "False" trong answer. Default fallback nếu không parse được.

### sem_extract → dicts (string values)

`extract_postprocess` (`postprocessors.py:149-179`):

1. JSON parsing: `json.loads(llm_answer)` (`postprocessors.py:170-171`)
2. Fallback: `{}` on failure (`postprocessors.py:173-174`)
3. **Tất cả values cast sang string**: `{key: str(value) for key, value in output.items()}` (`postprocessors.py:176`)

Điều này có nghĩa: kể cả khi LLM trả về `{"price": 29.99}` (number), output sẽ là `{"price": "29.99"}` (string).

### sem_topk → document index

`parse_ans_binary` (`sem_topk.py:83-129`):
```python
matches = list(re.finditer(r"Document[\s*](\d+)", answer, re.IGNORECASE))  # sem_topk.py:119
ans = int(matches[-1].group(1)) - 1  # sem_topk.py:122
```
Regex extract số document (1 or 2), trả về boolean (True = Document 1 thắng).

---

## 3. Không có Type Inference

LOTUS **KHÔNG** tự động infer types cho LLM output:

- Không convert string "29.99" → float
- Không convert string "2024-01-01" → datetime
- Không convert string "true" → boolean (ngoại trừ sem_filter)
- sem_extract luôn trả về `dict[str, str]` (`types.py:102`)

Nếu user cần typed data, phải tự convert sau:
```python
df["price"] = df["price"].astype(float)  # User phải tự làm
```

---

## 4. Type Definitions

**Location**: `types.py`

Các output types chính:
- `SemanticMapPostprocessOutput.outputs: list[str]` (`types.py:87-89`)
- `SemanticExtractPostprocessOutput.outputs: list[dict[str, str]]` (`types.py:101-103`)
- `SemanticFilterPostprocessOutput.outputs: list[bool]` (`types.py:114-116`)
- `SemanticTopKOutput.indexes: list[int]` (`types.py:179-181`)
- `SemanticAggOutput.outputs: list[str]` (`types.py:130-131`)

---

## 5. Kết luận

Hệ thống type của LOTUS có tính minimalist:
- Input: pandas dtypes + ImageDtype
- Output: strings, booleans, string dicts
- Không có type inference, không có automatic conversion
- Trade-off: đơn giản, predictable, nhưng user phải tự handle type conversion
