# A3 - Embedding Models

## RM Abstract Base Class

Định nghĩa tại `models/rm.py:10`:

```python
class RM(ABC):
    def __init__(self) -> None:
        pass

    @abstractmethod
    def _embed(self, docs: list[str]) -> NDArray[np.float64]:
        pass

    def __call__(self, docs: list[str]) -> NDArray[np.float64]:
        return self._embed(docs)

    def convert_query_to_query_vector(
        self,
        queries: Union[pd.Series, str, Image.Image, list[str], NDArray[np.float64]],
    ) -> NDArray[np.float64]:
```

### Interface chính:
- `_embed(docs)` (rm.py:27): Abstract method, subclass phải implement. Nhận list[str], trả về NDArray.
- `__call__(docs)` (rm.py:41): Public method, delegate to `_embed`.
- `convert_query_to_query_vector(queries)` (rm.py:53): Chuyển đổi nhiều dạng query thành embedding vectors:
  - `str` hoặc `Image.Image` -> wrap thành list, gọi `_embed`
  - `pd.Series` -> convert to list, gọi `_embed`
  - `list[str]` -> gọi `_embed` trực tiếp
  - `np.ndarray` -> trả về as-is (pre-computed vectors)

---

## Implementation 1: SentenceTransformersRM

Định nghĩa tại `models/sentence_transformers_rm.py:11`:

```python
class SentenceTransformersRM(RM):
    def __init__(
        self,
        model: str = "intfloat/e5-base-v2",
        max_batch_size: int = 64,
        normalize_embeddings: bool = True,
        device: str | None = None,
    ) -> None:
```

### Đặc điểm:
- **Default model**: `"intfloat/e5-base-v2"` (sentence_transformers_rm.py:28)
- **Local inference**: Sử dụng `SentenceTransformer` từ `sentence-transformers` package (sentence_transformers_rm.py:47)
- **Batch processing**: Xử lý `max_batch_size=64` documents mỗi lần (sentence_transformers_rm.py:67)
- **Normalization**: `normalize_embeddings=True` by default (sentence_transformers_rm.py:30)
- **GPU support**: Tự động sử dụng GPU nếu có, hoặc chỉ định `device="cuda"` (sentence_transformers_rm.py:31)

### _embed implementation (sentence_transformers_rm.py:49-76):
```python
def _embed(self, docs: list[str]) -> NDArray[np.float64]:
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):
        batch = docs[i : i + self.max_batch_size]
        _batch = convert_to_base_data(batch)
        torch_embeddings = self.transformer.encode(
            _batch, convert_to_tensor=True,
            normalize_embeddings=self.normalize_embeddings,
            show_progress_bar=False
        )
        cpu_embeddings = torch_embeddings.cpu().numpy()
        all_embeddings.append(cpu_embeddings)
    return np.vstack(all_embeddings)
```

Lưu ý: `convert_to_base_data(batch)` (sentence_transformers_rm.py:69) convert từ custom dtypes (như ImageDtype) sang base data.

---

## Implementation 2: LiteLLMRM

Định nghĩa tại `models/litellm_rm.py:11`:

```python
class LiteLLMRM(RM):
    def __init__(
        self,
        model: str = "text-embedding-3-small",
        max_batch_size: int = 64,
        truncate_limit: int | None = None,
    ) -> None:
```

### Đặc điểm:
- **Default model**: `"text-embedding-3-small"` (litellm_rm.py:27) - OpenAI embedding model
- **API-based**: Sử dụng `litellm.embedding()` (litellm_rm.py:68), tương tự như LM class sử dụng litellm cho completion
- **Provider switching**: Đổi model string để dùng provider khác (e.g. "cohere/embed-english-v3.0")
- **Truncation**: Optional `truncate_limit` để cắt văn bản dài (litellm_rm.py:65-66)

### _embed implementation (litellm_rm.py:45-71):
```python
def _embed(self, docs: list[str]) -> NDArray[np.float64]:
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):
        batch = docs[i : i + self.max_batch_size]
        if self.truncate_limit:
            batch = [doc[: self.truncate_limit] for doc in batch]
        _batch = convert_to_base_data(batch)
        response: EmbeddingResponse = embedding(model=self.model, input=_batch)
        embeddings = np.array([d["embedding"] for d in response.data])
        all_embeddings.append(embeddings)
    return np.vstack(all_embeddings)
```

---

## Implementation 3: ColBERTv2RM

Định nghĩa tại `models/colbertv2_rm.py:17`:

```python
class ColBERTv2RM:
    def __init__(self) -> None:
        self.docs: list[str] | None = None
        self.kwargs: dict[str, Any] = {"doc_maxlen": 300, "nbits": 2}
        self.index_dir: str | None = None
```

### Đặc biệt:
- **KHÔNG kế thừa RM**: `ColBERTv2RM` không extend `RM` ABC. Nó có interface riêng.
- **Built-in indexing**: `index()` method (colbertv2_rm.py:43) tạo index trực tiếp, không cần VS riêng
- **Checkpoint**: Sử dụng `"colbert-ir/colbertv2.0"` (colbertv2_rm.py:64)
- **Search interface**: `__call__(queries, K)` trả về `RMOutput` trực tiếp (colbertv2_rm.py:111)
- **get_vectors_from_index**: raise `NotImplementedError` (colbertv2_rm.py:109)
- **Conditional import**: ColBERT dependencies được import trong `try/except` (colbertv2_rm.py:10-14)

### index() method (colbertv2_rm.py:43):
```python
def index(self, docs: list[str], index_dir: str, **kwargs) -> None:
    checkpoint = "colbert-ir/colbertv2.0"
    with Run().context(RunConfig(nranks=1, experiment="lotus")):
        config = ColBERTConfig(doc_maxlen=kwargs["doc_maxlen"], nbits=kwargs["nbits"], kmeans_niters=4)
        indexer = Indexer(checkpoint=checkpoint, config=config)
        indexer.index(name=f"{index_dir}/index", collection=docs, overwrite=True)
    # Pickle docs separately
    with open(f"experiments/lotus/indexes/{index_dir}/index/docs", "wb") as fp:
        pickle.dump(docs, fp)
```

---

## So sánh 3 implementations

| Feature | SentenceTransformersRM | LiteLLMRM | ColBERTv2RM |
|---|---|---|---|
| Extends RM | Yes | Yes | **No** |
| Default model | `intfloat/e5-base-v2` | `text-embedding-3-small` | `colbert-ir/colbertv2.0` |
| Inference | Local (GPU/CPU) | API call | Local (GPU) |
| Batch size | 64 | 64 | N/A (batch by ColBERT) |
| Built-in index | No | No | Yes |
| Normalization | Yes (configurable) | Depends on model | N/A |
| File | sentence_transformers_rm.py:11 | litellm_rm.py:11 | colbertv2_rm.py:17 |
