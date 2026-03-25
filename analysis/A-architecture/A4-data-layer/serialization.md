# A4 - Serialization

## SerializationFormat Enum

Dinh nghia tai `types.py:206`:

```python
class SerializationFormat(Enum):
    JSON = "json"
    XML = "xml"
    DEFAULT = "default"
```

Cau hinh qua `lotus.settings.serialization_format` (settings.py:20):
```python
serialization_format: SerializationFormat = SerializationFormat.DEFAULT
```

## df2text Function (task_instructions.py:325)

Day la function chinh de serialize DataFrame rows thanh text strings cho LLM input.

```python
def df2text(df: pd.DataFrame, cols: list[str]) -> list[str]:
    """Formats the given DataFrame into a string containing info from cols."""

    def custom_format_row(x: pd.Series, cols: list[str]) -> str:
        return "".join([f"[{cols[i].capitalize()}]: \u00ab{x[cols[i]]}\u00bb\n" for i in range(len(cols))])

    def clean_and_escape_column_name(column_name: str) -> str:
        clean_name = re.sub(r"[^\w]", "", column_name)
        return clean_name

    cols = [col for col in cols if col in df.columns]
    if len(cols) == 0:
        return [""] * len(df)

    projected_df = df[cols]
    formatted_rows: list[str] = []

    if lotus.settings.serialization_format == SerializationFormat.DEFAULT:
        formatted_rows = projected_df.apply(lambda x: custom_format_row(x, cols), axis=1).tolist()
    elif lotus.settings.serialization_format == SerializationFormat.JSON:
        formatted_rows = projected_df.to_json(orient="records", lines=True).splitlines()
    elif lotus.settings.serialization_format == SerializationFormat.XML:
        ...
        projected_df = projected_df.rename(columns=lambda x: clean_and_escape_column_name(x))
        full_xml = projected_df.to_xml(root_name="data", row_name="row", pretty_print=False, index=False)
        root = ET.fromstring(full_xml)
        formatted_rows = [ET.tostring(row, encoding="unicode", method="xml") for row in root.findall("row")]

    return formatted_rows
```

## 3 Format Examples

Cho DataFrame row: `{"title": "AI Guide", "author": "John"}` va `cols=["title", "author"]`:

### DEFAULT (task_instructions.py:329, 343-344)

```
[Title]: <<AI Guide>>
[Author]: <<John>>
```

Format: `[{col.capitalize()}]: <<{value}>>\n` (guillemet characters `<<` va `>>`)

### JSON (task_instructions.py:345-346)

```json
{"title":"AI Guide","author":"John"}
```

Su dung `pandas.to_json(orient="records", lines=True)`. Moi row la 1 JSON object tren 1 dong.

### XML (task_instructions.py:347-359)

```xml
<row><title>AI Guide</title><author>John</author></row>
```

Su dung `pandas.to_xml()`. Column names duoc clean (loai bo special chars) truoc khi convert (task_instructions.py:356).

## df2multimodal_info (task_instructions.py:364)

Function nay wrap `df2text` va them image data:

```python
def df2multimodal_info(df: pd.DataFrame, cols: list[str]) -> list[dict[str, Any]]:
    image_cols = [col for col in cols if isinstance(df[col].dtype, ImageDtype)]
    text_cols = [col for col in cols if col not in image_cols]
    text_rows = df2text(df, text_cols)
    multimodal_data = [
        {
            "text": text_rows[i],
            "image": {col.capitalize(): df[col].array.get_image(i, "base64") for col in image_cols},
        }
        for i in range(len(df))
    ]
    return multimodal_data
```

Output format cho moi row:
```python
{
    "text": "[Title]: <<AI Guide>>\n[Author]: <<John>>\n",
    "image": {}  # hoac {"Photo": "data:image/png;base64,..."} neu co image column
}
```

## context_formatter (task_instructions.py:40)

Convert multimodal_data thanh (text, image_inputs) cho LLM messages:

```python
def context_formatter(multimodal_data):
    if isinstance(multimodal_data, str):
        text = multimodal_data
        image_inputs = []
    elif isinstance(multimodal_data, dict):
        image_data = multimodal_data.get("image", {})
        _image_inputs = [
            (
                {"type": "text", "text": f"[{key.capitalize()}]: \n"},
                {"type": "image_url", "image_url": {"url": base64_image}},
            )
            for key, base64_image in image_data.items()
        ]
        image_inputs = [m for image_input in _image_inputs for m in image_input]
        text = multimodal_data["text"] or ""
    return text, image_inputs
```

## user_message_formatter (task_instructions.py:68)

Tao user message dict cho LLM:

```python
def user_message_formatter(multimodal_data, user_instruction_with_tag=None):
    text, image_inputs = context_formatter(multimodal_data)
    if not image_inputs:
        return {
            "role": "user",
            "content": f"Context:\n{text}\n\n{user_instruction_with_tag}",
        }
    content = [{"type": "text", "text": f"Context:\n{text}"}] + image_inputs
    if user_instruction_with_tag:
        content.append({"type": "text", "text": f"\n\n{user_instruction_with_tag}"})
    return {"role": "user", "content": content}
```

- Text-only: `content` la string don gian
- Multimodal: `content` la list of content parts (OpenAI vision format)

## merge_multimodal_info (task_instructions.py:382)

Dung trong `sem_join` de ket hop 2 rows:

```python
def merge_multimodal_info(first, second):
    return [
        {
            "text": f"{first[i]['text']}\n{second[j]['text']}"
                if first[i]["text"] != "" and second[j]["text"] != ""
                else first[i]["text"] + second[j]["text"],
            "image": {**first[i]["image"], **second[j]["image"]},
        }
        for i in range(len(first))
        for j in range(len(second))
    ]
```

Tao cross product: moi row cua first duoc merge voi moi row cua second. Text duoc noi bang newline, images duoc merge.

## li2text (task_instructions.py:405)

```python
def li2text(li: list[str], name: str) -> str:
    return "".join([f"[{name}] {li[i]}\n" for i in range(len(li))])
```

Format list thanh text voi label prefix. Dung trong aggregation.
