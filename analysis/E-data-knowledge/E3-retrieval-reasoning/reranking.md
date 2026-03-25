# E3 - Reranking

## Tổng quan

LOTUS hỗ trợ two-stage retrieval: vector search (recall) → cross-encoder reranking (precision). Reranking là optional, chỉ khi user cung cấp `n_rerank` parameter.

---

## 1. Reranker Abstract Class

**Location**: `models/reranker.py:6-24`

```python
class Reranker(ABC):
    @abstractmethod
    def __call__(self, query: str, docs: list[str], K: int) -> RerankerOutput:
```

Interface đơn giản:
- Input: query string, list of documents, K (số docs cần giữ)
- Output: `RerankerOutput` chứa indices of reranked documents (`types.py:199-200`)

---

## 2. CrossEncoderReranker

**Location**: `models/cross_encoder_reranker.py:7-59`

```python
class CrossEncoderReranker(Reranker):
    def __init__(
        self,
        model="mixedbread-ai/mxbai-rerank-large-v1",   # :20
        device=None,                                      # :21
        max_batch_size=64,                                 # :22
    ):
        self.max_batch_size = max_batch_size               # :35
        self.model = CrossEncoder(model, device=device)    # :36
```

### Default model
`"mixedbread-ai/mxbai-rerank-large-v1"` - state-of-the-art cross-encoder reranker.

### Reranking implementation (`cross_encoder_reranker.py:38-59`)
```python
def __call__(self, query, docs, K):
    results = self.model.rank(
        query, docs, top_k=K,
        batch_size=self.max_batch_size,                   # :57
        show_progress_bar=False
    )
    indices = [int(result["corpus_id"]) for result in results]  # :58
    return RerankerOutput(indices=indices)                       # :59
```

Dùng `CrossEncoder.rank()` từ sentence-transformers - internally:
1. Tạo (query, doc) pairs cho tất cả documents
2. Score mỗi pair bằng cross-encoder model
3. Sort by score, trả về top-K

---

## 3. Integration trong sem_search

**Location**: `sem_search.py:148-155`

```python
if n_rerank is not None:
    if lotus.settings.reranker is None:                      # :149
        raise ValueError("Reranker not found in settings")   # :150
    docs = new_df[col_name].tolist()                          # :152
    reranked_output = lotus.settings.reranker(query, docs, n_rerank)  # :153
    reranked_idxs = reranked_output.indices                   # :154
    new_df = new_df.iloc[reranked_idxs]                       # :155
```

### Two-stage pipeline

1. **Stage 1 - Vector Retrieval**: Lấy K documents qua FAISS
2. **Stage 2 - Cross-Encoder Reranking**: Rerank K documents, giữ top `n_rerank`

Ví dụ:
```python
df.sem_search('title', 'AI tutorials', K=100, n_rerank=10)
# Stage 1: FAISS retrieves top 100
# Stage 2: Cross-encoder reranks 100 → keeps top 10
```

---

## 4. Configuration

Reranker được configure qua `lotus.settings`:

```python
lotus.settings.configure(
    reranker=CrossEncoderReranker(
        model="mixedbread-ai/mxbai-rerank-large-v1",
        device="cuda"
    )
)
```

---

## 5. Khi nào dùng reranking

- **K only**: Chỉ vector search - nhanh, approximate
- **K + n_rerank**: Vector search + reranking - chậm hơn, chính xác hơn
- **n_rerank only**: Không có K → rerank toàn bộ DataFrame (assert fails: `sem_search.py:101`)

Thực tế: `K` hoặc `n_rerank` phải được provided (`sem_search.py:101`):
```python
assert not (K is None and n_rerank is None), "K or n_rerank must be provided"
```

---

## 6. Kết luận

Reranking trong LOTUS:
- Optional two-stage pipeline
- CrossEncoder default: mixedbread-ai/mxbai-rerank-large-v1
- Simple integration: vector search → rerank → top-n_rerank
- Extensible: custom rerankers qua abstract Reranker class
