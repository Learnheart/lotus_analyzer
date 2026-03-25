# A3 - Vector Index (Vector Store)

## VS Abstract Base Class

Dinh nghia tai `vector_store/vs.py:10`:

```python
class VS(ABC):
    def __init__(self) -> None:
        self.index_dir: str | None = None

    @abstractmethod
    def index(self, docs: list[str], embeddings: NDArray[np.float64], index_dir: str, **kwargs):
        pass

    @abstractmethod
    def load_index(self, index_dir: str):
        pass

    @abstractmethod
    def __call__(
        self,
        query_vectors: NDArray[np.float64],
        K: int,
        ids: list[int] | None = None,
        **kwargs: dict[str, Any],
    ) -> RMOutput:
        pass

    @abstractmethod
    def get_vectors_from_index(self, index_dir: str, ids: list[int]) -> NDArray[np.float64]:
        pass
```

### Interface:
| Method | Muc dich | File:Line |
|---|---|---|
| `index(docs, embeddings, index_dir)` | Tao va luu index | vs.py:17 |
| `load_index(index_dir)` | Load index tu disk | vs.py:24 |
| `__call__(query_vectors, K, ids)` | Nearest neighbor search | vs.py:31 |
| `get_vectors_from_index(index_dir, ids)` | Lay vectors theo ID | vs.py:54 |

---

## FaissVS Implementation

Dinh nghia tai `vector_store/faiss_vs.py:13`:

```python
class FaissVS(VS):
    def __init__(self, factory_string: str = "Flat", metric=faiss.METRIC_INNER_PRODUCT):
        super().__init__()
        self.factory_string = factory_string
        self.metric = metric
        self.index_dir: str | None = None
        self.faiss_index: faiss.Index | None = None
        self.vecs: NDArray[np.float64] | None = None
```

### Constructor parameters:
- `factory_string`: Default `"Flat"` - brute-force exact search. Co the doi thanh `"IVF100,Flat"` cho approximate search.
- `metric`: Default `faiss.METRIC_INNER_PRODUCT` - cosine similarity (khi vectors da normalized)

### index() method (faiss_vs.py:22-30):

```python
def index(self, docs, embeddings, index_dir, **kwargs):
    self.faiss_index = faiss.index_factory(embeddings.shape[1], self.factory_string, self.metric)
    self.faiss_index.add(embeddings)
    self.index_dir = index_dir

    os.makedirs(index_dir, exist_ok=True)
    with open(f"{index_dir}/vecs", "wb") as fp:
        pickle.dump(embeddings, fp)
    faiss.write_index(self.faiss_index, f"{index_dir}/index")
```

Luu 2 files:
1. `{index_dir}/vecs`: pickle dump cua embeddings NDArray (faiss_vs.py:28-29)
2. `{index_dir}/index`: faiss native index file (faiss_vs.py:30)

### load_index() method (faiss_vs.py:32-36):

```python
def load_index(self, index_dir: str) -> None:
    self.index_dir = index_dir
    self.faiss_index = faiss.read_index(f"{index_dir}/index")
    with open(f"{index_dir}/vecs", "rb") as fp:
        self.vecs = pickle.load(fp)
```

Load ca faiss index va raw vectors tu disk.

### __call__() method (faiss_vs.py:43-77):

```python
def __call__(self, query_vectors, K, ids=None, **kwargs) -> RMOutput:
    if self.faiss_index is None or self.index_dir is None:
        raise ValueError("Index not loaded")

    if ids is not None:
        # Subset search: tao temporary index chi voi vectors cua ids duoc chi dinh
        subset_vecs = self.get_vectors_from_index(self.index_dir, ids)
        tmp_index = faiss.index_factory(subset_vecs.shape[1], self.factory_string, self.metric)
        tmp_index.add(subset_vecs)
        distances, sub_indices = tmp_index.search(query_vectors, K)
        # Remap sub-indices -> original global ids
        subset_ids = np.array(ids)
        indices = np.array([subset_ids[sub_indices[i]] for i in range(len(sub_indices))]).tolist()
    else:
        distances, indices = self.faiss_index.search(query_vectors, K)

    return RMOutput(distances=distances, indices=indices)
```

Dac biet: khi `ids` duoc cung cap (subset search):
1. Lay vectors cua subset tu disk (faiss_vs.py:59)
2. Tao temporary faiss index (faiss_vs.py:63-64)
3. Search tren temporary index (faiss_vs.py:67)
4. Map lai sub-indices thanh global indices (faiss_vs.py:71-72)

Day la co che hieu qua de search tren subset cua DataFrame (vi du khi DataFrame da duoc filter truoc).

### get_vectors_from_index (faiss_vs.py:38-41):

```python
def get_vectors_from_index(self, index_dir: str, ids: list[int]) -> NDArray[np.float64]:
    with open(f"{index_dir}/vecs", "rb") as fp:
        vecs = pickle.load(fp)
    return vecs[ids]
```

Load toan bo vectors tu pickle file, roi numpy index theo ids.

## Luu y

1. **Flat index**: Default `"Flat"` cho exact search, phu hop voi datasets nho-vua. Doi voi datasets lon, nen dung index khac nhu `"IVF100,Flat"`.

2. **METRIC_INNER_PRODUCT**: Gia dinh vectors da duoc normalize (SentenceTransformersRM default `normalize_embeddings=True`). Inner product cua normalized vectors = cosine similarity.

3. **Pickle cho vectors**: Raw vectors luon duoc pickle dump rieng (faiss_vs.py:28-29). Dieu nay cho phep `get_vectors_from_index` lay vectors theo ID ma khong can reconstruct tu faiss index.

4. **Khong co in-memory cache**: Moi lan `load_index` load tu disk. `sem_search` kiem tra `vs.index_dir != col_index_dir` (sem_search.py:112-113) de tranh load lai cung index.
