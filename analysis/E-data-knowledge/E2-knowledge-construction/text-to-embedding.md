# E2 - Text to Embedding Pipeline

## Tổng quan

LOTUS chuyển đổi raw text thành vector embeddings thông qua RM (Retrieval Model) classes. Pipeline: raw text → RM._embed() → numpy array → VS.index() → stored index.

---

## 1. Pipeline tổng thể

```
df[col_name].tolist() → rm(docs) → rm._embed(docs) → NDArray[np.float64] → vs.index(docs, embeddings, index_dir)
```

Trigger point: `sem_index` accessor (`sem_index.py:62-77`):
```python
embeddings = rm(self._obj[col_name].tolist())    # sem_index.py:74
vs.index(self._obj[col_name], embeddings, index_dir)  # sem_index.py:75
```

---

## 2. SentenceTransformersRM

**Location**: `sentence_transformers_rm.py:11-76`

### Configuration
- Default model: `"intfloat/e5-base-v2"` (`sentence_transformers_rm.py:28`)
- `max_batch_size: int = 64` (`sentence_transformers_rm.py:29`)
- `normalize_embeddings: bool = True` (`sentence_transformers_rm.py:30`)
- Device support: `device: str | None = None` (`sentence_transformers_rm.py:31`)

### Embedding process (`sentence_transformers_rm.py:49-76`)
```python
def _embed(self, docs: list[str]) -> NDArray[np.float64]:
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):    # :67
        batch = docs[i : i + self.max_batch_size]                # :68
        _batch = convert_to_base_data(batch)                     # :69
        torch_embeddings = self.transformer.encode(
            _batch, convert_to_tensor=True,
            normalize_embeddings=self.normalize_embeddings,      # :71
            show_progress_bar=False
        )
        cpu_embeddings = torch_embeddings.cpu().numpy()          # :74
        all_embeddings.append(cpu_embeddings)
    return np.vstack(all_embeddings)                             # :76
```

**Đặc điểm**:
- Batch processing: xử lý `max_batch_size=64` docs mỗi lần
- `normalize_embeddings=True` mặc định → cosine similarity khi dùng inner product
- Convert tensor sang numpy trên CPU
- `convert_to_base_data` chuyển đổi ImageArray data

---

## 3. LiteLLMRM

**Location**: `litellm_rm.py:11-71`

### Configuration
- Default model: `"text-embedding-3-small"` (`litellm_rm.py:27`)
- `max_batch_size: int = 64` (`litellm_rm.py:28`)
- `truncate_limit: int | None = None` (`litellm_rm.py:29`)

### Embedding process (`litellm_rm.py:45-71`)
```python
def _embed(self, docs: list[str]) -> NDArray[np.float64]:
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):  # :63
        batch = docs[i : i + self.max_batch_size]              # :64
        if self.truncate_limit:
            batch = [doc[:self.truncate_limit] for doc in batch]  # :66 - character truncation
        _batch = convert_to_base_data(batch)                   # :67
        response = embedding(model=self.model, input=_batch)   # :68
        embeddings = np.array([d["embedding"] for d in response.data])  # :69
        all_embeddings.append(embeddings)
    return np.vstack(all_embeddings)                           # :71
```

**Đặc điểm**:
- Dùng `litellm.embedding()` API → hỗ trợ multiple providers (OpenAI, Cohere, etc.)
- Optional `truncate_limit`: cắt text theo **characters** (không phải tokens)
- Response format: list of dicts với key "embedding"

---

## 4. Không có chunking cho embedding

**Quan trọng**: LOTUS embed toàn bộ cell value cùng lúc - không split thành chunks.

- Mỗi cell = 1 embedding vector
- Nếu cell quá dài, chỉ có `truncate_limit` của LiteLLMRM để cắt
- SentenceTransformersRM dựa vào model's max_seq_length tự động truncate
- Không có overlap giữa chunks (vì không có chunking)

---

## 5. Index Storage

Sau khi embed, vectors được lưu qua VS (Vector Store):

```python
vs.index(self._obj[col_name], embeddings, index_dir)  # sem_index.py:75
```

Index directory được lưu trong DataFrame attrs:
```python
self._obj.attrs["index_dirs"][col_name] = index_dir  # sem_index.py:76
```

Chi tiết storage xem file `indexing-strategies.md`.
