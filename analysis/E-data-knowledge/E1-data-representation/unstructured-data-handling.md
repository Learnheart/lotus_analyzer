# E1 - Unstructured Data Handling

## Tổng quan

LOTUS xử lý unstructured text columns tương đối "thô" - không preprocessing, không chunking ở row level. Các chiến lược long context chỉ được áp dụng trong sem_agg khi tổng documents vượt quá context window.

---

## 1. Text columns: Không preprocessing

Text columns được serialize nguyên trạng qua `df2text` (`task_instructions.py:325-361`). Không có:
- Tokenization
- Stopword removal
- Stemming/Lemmatization
- Chunking ở row level (mỗi row = 1 unit)

Mỗi cell value được nhúng trực tiếp vào prompt:
```
[Column_name]: «raw cell value»
```

---

## 2. Long Context Strategies

**Location**: `long_context_strategy.py:57-82`

```python
def create_chunked_documents(
    df, cols, model, strategy, extra_tokens
) -> ChunkedDocument:
```

### 2.1 TRUNCATE Strategy

**Location**: `long_context_strategy.py:85-142`

Khi document vượt quá `max_doc_tokens = model.max_ctx_len - model.max_tokens - extra_tokens` (`long_context_strategy.py:96`):

1. Tính available tokens sau khi trừ ellipsis: `available_tokens = max_doc_tokens - ellipsis_tokens` (`long_context_strategy.py:119`)
2. Encode text thành tokens: `tokens = model.encode_text(doc_str)` (`long_context_strategy.py:126`)
3. Cắt tokens: `truncated_tokens = tokens[:available_tokens]` (`long_context_strategy.py:129`)
4. Decode lại thành text và thêm "...": `truncated_text + ellipsis` (`long_context_strategy.py:133`)

### 2.2 CHUNK Strategy

**Location**: `long_context_strategy.py:145-233`

Chiến lược thông minh hơn - tìm column có nhiều tokens nhất và split column đó:

1. Đếm tokens mỗi column: `col_tokens = model.count_tokens(col_content)` (`long_context_strategy.py:181`)
2. Tìm column lớn nhất: `max_tokens_col` (`long_context_strategy.py:183-185`)
3. Tính available tokens cho column lớn nhất: `available_tokens = max_doc_tokens - doc_str_emptied_tokens` (`long_context_strategy.py:197`)
4. Split column lớn nhất bằng `_split_text_by_tokens` (`long_context_strategy.py:210`)
5. Duplicate các columns khác cho mỗi chunk (`long_context_strategy.py:213-226`)

---

## 3. Token-based Splitting

**Location**: `long_context_strategy.py:236-261`

```python
def _split_text_by_tokens(text, max_tokens, model) -> list[str]:
```

- Encode toàn bộ text thành tokens: `tokens = model.encode_text(text)` (`long_context_strategy.py:249`)
- Split tokens thành chunks kích thước `max_tokens`: `chunk_tokens = tokens[i : i + max_tokens]` (`long_context_strategy.py:257`)
- Decode mỗi chunk: `chunk_text = model.decode_tokens(chunk_tokens)` (`long_context_strategy.py:258`)

**Lưu ý**: Splitting theo token boundary, không theo sentence/paragraph boundary. Có thể cắt giữa từ/câu.

---

## 4. ChunkedDocument class

**Location**: `long_context_strategy.py:22-55`

```python
@dataclass
class ChunkedDocument:
    strategy: LongContextStrategy
    docs: list[str]
    chunk_info: list[ChunkInfo]
    original_df: pd.DataFrame
```

- `ChunkInfo` lưu `original_row_idx`, `chunk_idx`, `total_chunks`, `chunked_column` (`long_context_strategy.py:12-18`)
- Hỗ trợ truy ngược về row gốc: `get_row(index)` (`long_context_strategy.py:45-50`)
- `get_value(index, column)` lấy giá trị từ original DataFrame (`long_context_strategy.py:52-54`)

---

## 5. LiteLLMRM truncation cho embeddings

**Location**: `litellm_rm.py:29`

```python
truncate_limit: int | None = None
```

Khi `truncate_limit` được set, text được cắt theo character count trước khi embedding:
```python
batch = [doc[:self.truncate_limit] for doc in batch]  # litellm_rm.py:66
```

Đây là truncation theo **characters**, không phải tokens - khác với long_context_strategy dùng token-based truncation.

---

## 6. Kết luận

LOTUS có hai tầng xử lý long content:
1. **Row level** (embedding): Character-based truncation trong LiteLLMRM
2. **Aggregation level** (sem_agg): Token-based TRUNCATE hoặc CHUNK strategies

Không có chunking ở tầng individual row cho LLM operations (sem_filter, sem_map, etc.) - toàn bộ cell value được gửi trong prompt.
