# A3 - Embedding Models

## RM Abstract Base Class

Dinh nghia tai `models/rm.py:10`:

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

### Interface chinh:
- `_embed(docs)` (rm.py:27): Abstract method, subclass phai implement. Nhan list[str], tra ve NDArray.
- `__call__(docs)` (rm.py:41): Public method, delegate to `_embed`.
- `convert_query_to_query_vector(queries)` (rm.py:53): Chuyen doi nhieu dang query thanh embedding vectors:
  - `str` hoac `Image.Image` -> wrap thanh list, goi `_embed`
  - `pd.Series` -> convert to list, goi `_embed`
  - `list[str]` -> goi `_embed` truc tiep
  - `np.ndarray` -> tra ve as-is (pre-computed vectors)

---

## Implementation 1: SentenceTransformersRM

Dinh nghia tai `models/sentence_transformers_rm.py:11`:

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

### Dac diem:
- **Default model**: `"intfloat/e5-base-v2"` (sentence_transformers_rm.py:28)
- **Local inference**: Su dung `SentenceTransformer` tu `sentence-transformers` package (sentence_transformers_rm.py:47)
- **Batch processing**: Xu ly `max_batch_size=64` documents moi lan (sentence_transformers_rm.py:67)
- **Normalization**: `normalize_embeddings=True` by default (sentence_transformers_rm.py:30)
- **GPU support**: Tu dong su dung GPU neu co, hoac chi dinh `device="cuda"` (sentence_transformers_rm.py:31)

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

Luu y: `convert_to_base_data(batch)` (sentence_transformers_rm.py:69) convert tu custom dtypes (nhu ImageDtype) sang base data.

---

## Implementation 2: LiteLLMRM

Dinh nghia tai `models/litellm_rm.py:11`:

```python
class LiteLLMRM(RM):
    def __init__(
        self,
        model: str = "text-embedding-3-small",
        max_batch_size: int = 64,
        truncate_limit: int | None = None,
    ) -> None:
```

### Dac diem:
- **Default model**: `"text-embedding-3-small"` (litellm_rm.py:27) - OpenAI embedding model
- **API-based**: Su dung `litellm.embedding()` (litellm_rm.py:68), tuong tu nhu LM class su dung litellm cho completion
- **Provider switching**: Doi model string de dung provider khac (e.g. "cohere/embed-english-v3.0")
- **Truncation**: Optional `truncate_limit` de cat van ban dai (litellm_rm.py:65-66)

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

Dinh nghia tai `models/colbertv2_rm.py:17`:

```python
class ColBERTv2RM:
    def __init__(self) -> None:
        self.docs: list[str] | None = None
        self.kwargs: dict[str, Any] = {"doc_maxlen": 300, "nbits": 2}
        self.index_dir: str | None = None
```

### Dac biet:
- **KHONG ke thua RM**: `ColBERTv2RM` khong extend `RM` ABC. No co interface rieng.
- **Built-in indexing**: `index()` method (colbertv2_rm.py:43) tao index truc tiep, khong can VS rieng
- **Checkpoint**: Su dung `"colbert-ir/colbertv2.0"` (colbertv2_rm.py:64)
- **Search interface**: `__call__(queries, K)` tra ve `RMOutput` truc tiep (colbertv2_rm.py:111)
- **get_vectors_from_index**: raise `NotImplementedError` (colbertv2_rm.py:109)
- **Conditional import**: ColBERT dependencies duoc import trong `try/except` (colbertv2_rm.py:10-14)

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

## So sanh 3 implementations

| Feature | SentenceTransformersRM | LiteLLMRM | ColBERTv2RM |
|---|---|---|---|
| Extends RM | Yes | Yes | **No** |
| Default model | `intfloat/e5-base-v2` | `text-embedding-3-small` | `colbert-ir/colbertv2.0` |
| Inference | Local (GPU/CPU) | API call | Local (GPU) |
| Batch size | 64 | 64 | N/A (batch by ColBERT) |
| Built-in index | No | No | Yes |
| Normalization | Yes (configurable) | Depends on model | N/A |
| File | sentence_transformers_rm.py:11 | litellm_rm.py:11 | colbertv2_rm.py:17 |
