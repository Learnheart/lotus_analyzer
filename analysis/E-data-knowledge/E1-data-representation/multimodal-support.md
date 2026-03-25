# E1 - Multimodal Support

## Tổng quan

LOTUS hỗ trợ multimodal data (text + images) thông qua custom `ImageDtype`/`ImageArray` và cơ chế serialization riêng cho image columns.

---

## 1. ImageDtype + ImageArray

### ImageDtype

**Location**: `dtype_extensions/image.py:12-35`

```python
class ImageDtype(ExtensionDtype):
    name = "image"
    type = Image.Image
    na_value = None
```

### ImageArray

**Location**: `dtype_extensions/image.py:37-305`

- Lưu trữ image references (paths, URLs, base64 strings, PIL Images, numpy arrays) trong `_data: np.ndarray` (`image.py:58`)
- Cache mechanism: `_cached_images: dict[tuple[int, str], str | Image.Image | None]` (`image.py:61`)
- Lazy loading: images chỉ được fetch khi cần qua `get_image()` (`image.py:117-132`)
- Hỗ trợ 2 output types: `"Image"` (PIL Image) và `"base64"` (base64 string) (`image.py:60`)

---

## 2. df2multimodal_info - Tách text và image

**Location**: `task_instructions.py:364-379`

```python
def df2multimodal_info(df, cols) -> list[dict[str, Any]]:
    image_cols = [col for col in cols if isinstance(df[col].dtype, ImageDtype)]  # :369
    text_cols = [col for col in cols if col not in image_cols]                    # :370
    text_rows = df2text(df, text_cols)                                           # :371
    multimodal_data = [
        {
            "text": text_rows[i],
            "image": {col.capitalize(): df[col].array.get_image(i, "base64") for col in image_cols},  # :375
        }
        for i in range(len(df))
    ]
```

Quá trình:
1. Phân loại columns thành `image_cols` và `text_cols` dựa trên dtype (`task_instructions.py:369-370`)
2. Text columns serialize qua `df2text` như bình thường (`task_instructions.py:371`)
3. Image columns được convert sang base64 qua `get_image(i, "base64")` (`task_instructions.py:375`)
4. Output: list of dicts, mỗi dict có key `"text"` và `"image"`

---

## 3. user_message_formatter - Tạo multimodal messages

**Location**: `task_instructions.py:68-84`

```python
def user_message_formatter(multimodal_data, user_instruction_with_tag=None):
    text, image_inputs = context_formatter(multimodal_data)       # :72
    if not image_inputs or len(image_inputs) == 0:                # :73
        return {"role": "user", "content": f"Context:\n{text}..."}
    content = [{"type": "text", "text": f"Context:\n{text}"}] + image_inputs  # :78
```

Khi có images, message sử dụng format **multipart content** với `image_url` entries:

### context_formatter

**Location**: `task_instructions.py:40-65`

```python
_image_inputs = [
    (
        {"type": "text", "text": f"[{key.capitalize()}]: \n"},         # :52
        {"type": "image_url", "image_url": {"url": base64_image}},     # :55-56
    )
    for key, base64_image in image_data.items()
]
```

Mỗi image tạo 2 entries:
1. Text label: `[Column_name]:`
2. Image URL: `{"type": "image_url", "image_url": {"url": "data:image/png;base64,..."}}`

---

## 4. fetch_image - Universal Image Loader

**Location**: `utils.py:75-120`

Hỗ trợ nhiều input types:

| Input | Handler | Line |
|---|---|---|
| `PIL.Image.Image` | Trả về trực tiếp | `utils.py:80-81` |
| `numpy.ndarray` | `Image.fromarray(image.astype("uint8"))` | `utils.py:82-83` |
| `http://` / `https://` | `Image.open(requests.get(image, stream=True).raw)` | `utils.py:84-85` |
| `file://` | `Image.open(image[7:])` | `utils.py:86-87` |
| `data:image` base64 | Decode base64, `Image.open(BytesIO(data))` | `utils.py:88-92` |
| `s3://` | `boto3` S3 client, `Image.open(BytesIO(image_data))` | `utils.py:93-107` |
| Other (local path) | `Image.open(image)` | `utils.py:109` |

Khi output type là `"base64"` (`utils.py:115-118`):
```python
image_obj = image_obj.convert("RGB")
buffered = BytesIO()
image_obj.save(buffered, format="PNG")
return "data:image/png;base64," + base64.b64encode(buffered.getvalue()).decode("utf-8")
```

Tất cả images được convert sang RGB trước khi encoding (`utils.py:114`).

---

## 5. merge_multimodal_info cho sem_join

**Location**: `task_instructions.py:382-402`

```python
def merge_multimodal_info(first, second):
    return [
        {
            "text": f"{first[i]['text']}\n{second[j]['text']}",       # :395-397
            "image": {**first[i]["image"], **second[j]["image"]},     # :398
        }
        for i in range(len(first))
        for j in range(len(second))
    ]
```

Kết hợp text bằng newline, merge image dicts. Tạo cartesian product (M x N).

---

## 6. Kết luận

Multimodal support trong LOTUS khá toàn diện:
- Custom dtype cho image columns
- Automatic separation text/image trong serialization
- Base64 encoding cho LLM API compatibility
- Universal image loader hỗ trợ nhiều sources
- Cartesian merge cho join operations
