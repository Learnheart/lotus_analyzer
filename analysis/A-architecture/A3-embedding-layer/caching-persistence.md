# A3 - Caching & Persistence for Vector Index

## Index Persistence

### FaissVS.index() - Lưu index (faiss_vs.py:22-30)

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

Tạo 2 files trên disk:
1. **`{index_dir}/vecs`** (faiss_vs.py:28-29): `pickle.dump(embeddings)` - raw NDArray embeddings
2. **`{index_dir}/index`** (faiss_vs.py:30): `faiss.write_index()` - faiss native binary format

### FaissVS.load_index() - Load index (faiss_vs.py:32-36)

```python
def load_index(self, index_dir: str) -> None:
    self.index_dir = index_dir
    self.faiss_index = faiss.read_index(f"{index_dir}/index")
    with open(f"{index_dir}/vecs", "rb") as fp:
        self.vecs = pickle.load(fp)
```

Load cả 2 files vào memory: faiss index và raw vectors.

### Tại sao cần cả 2 files?

- **faiss index**: Cần cho `search()` operation. Faiss không cho phép truy cập trực tiếp vector theo ID.
- **pickle vecs**: Cần cho `get_vectors_from_index()` (faiss_vs.py:38-41) để lấy vectors theo indices. Được dùng trong:
  - `sem_sim_join` khi load query embeddings từ index (sem_sim_join.py:114-117)
  - `FaissVS.__call__` khi search trên subset (faiss_vs.py:59)

---

## sem_index - Tạo index (sem_index.py:9)

### SemIndexDataframe.__call__ (sem_index.py:62-77)

```python
@operator_cache
def __call__(self, col_name: str, index_dir: str) -> pd.DataFrame:
    rm = lotus.settings.rm
    vs = lotus.settings.vs
    ...
    embeddings = rm(self._obj[col_name].tolist())      # Tạo embeddings
    vs.index(self._obj[col_name], embeddings, index_dir) # Lưu index
    self._obj.attrs["index_dirs"][col_name] = index_dir  # Ghi nhớ index_dir
    return self._obj
```

Flow:
1. Lấy RM và VS từ settings
2. Gọi `rm(docs)` để tạo embeddings cho tất cả documents
3. Gọi `vs.index(docs, embeddings, index_dir)` để lưu
4. Lưu `index_dir` vào `attrs["index_dirs"][col_name]`

### Lưu ý: __init__ reset attrs (sem_index.py:54)

```python
def __init__(self, pandas_obj: Any) -> None:
    self._validate(pandas_obj)
    self._obj = pandas_obj
    self._obj.attrs["index_dirs"] = {}  # RESET mỗi lần!
```

**Cảnh báo**: Mỗi lần truy cập `df.sem_index`, `__init__` được gọi và reset `attrs["index_dirs"]` thành `{}`. Điều này có nghĩa chỉ có thể dùng `sem_index` 1 lần hoặc phải chain calls.

---

## load_sem_index - Load index (load_sem_index.py:6)

### LoadSemIndexDataframe.__call__ (load_sem_index.py:49-51)

```python
def __call__(self, col_name: str, index_dir: str) -> pd.DataFrame:
    self._obj.attrs["index_dirs"][col_name] = index_dir
    return self._obj
```

**Quan trọng**: `load_sem_index` CHỈ set metadata `attrs["index_dirs"]`. Nó KHÔNG thực sự load index vào memory. Index được load lazily khi operator cần (ví dụ `sem_search` gọi `vs.load_index()` tại sem_search.py:113).

### Tương tự sem_index, __init__ reset attrs (load_sem_index.py:42):
```python
def __init__(self, pandas_obj: Any):
    self._validate(pandas_obj)
    self._obj = pandas_obj
    self._obj.attrs["index_dirs"] = {}
```

---

## Lazy Loading trong sem_search (sem_search.py:111-114)

```python
col_index_dir = self._obj.attrs["index_dirs"][col_name]
if vs.index_dir != col_index_dir:
    vs.load_index(col_index_dir)
assert vs.index_dir == col_index_dir
```

- Kiểm tra `vs.index_dir` có khớp với index cần dùng không
- Chỉ load khi cần thiết (index khác với index hiện tại)
- VS là singleton global (qua `lotus.settings.vs`), nên chỉ có 1 index được load tại 1 thời điểm

### Hạn chế: Chỉ 1 index tại 1 thời điểm

Vì `lotus.settings.vs` là shared singleton, khi 2 operations cần 2 index khác nhau, phải load/unload liên tục. Ví dụ:

```python
df1.sem_search("col_a", "query1", K=5)  # Load col_a index
df2.sem_search("col_b", "query2", K=5)  # Load col_b index (unload col_a)
df1.sem_search("col_a", "query3", K=5)  # Load col_a index AGAIN
```

---

## ColBERTv2RM Persistence (colbertv2_rm.py:43-93)

ColBERTv2RM có persistence riêng, không dùng VS:

### index (colbertv2_rm.py:66-72):
```python
with Run().context(RunConfig(nranks=1, experiment="lotus")):
    config = ColBERTConfig(...)
    indexer = Indexer(checkpoint=checkpoint, config=config)
    indexer.index(name=f"{index_dir}/index", collection=docs, overwrite=True)

with open(f"experiments/lotus/indexes/{index_dir}/index/docs", "wb") as fp:
    pickle.dump(docs, fp)
```

### load_index (colbertv2_rm.py:91-93):
```python
self.index_dir = index_dir
with open(f"experiments/lotus/indexes/{index_dir}/index/docs", "rb") as fp:
    self.docs = pickle.load(fp)
```

ColBERT lưu index tại `experiments/lotus/indexes/` (hardcoded path).

---

## Index Propagation

Khi DataFrame được filter hoặc transform, `index_dirs` attrs được propagate:

```python
# sem_filter.py:564
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)

# sem_search.py:141
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)
```

Điều này cho phép chain operations: `df.sem_search(...).sem_filter(...)` mà vẫn giữ được index references.
