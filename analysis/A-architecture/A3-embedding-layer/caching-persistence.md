# A3 - Caching & Persistence for Vector Index

## Index Persistence

### FaissVS.index() - Luu index (faiss_vs.py:22-30)

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

Tao 2 files tren disk:
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

Load ca 2 files vao memory: faiss index va raw vectors.

### Tai sao can ca 2 files?

- **faiss index**: Can cho `search()` operation. Faiss khong cho phep truy cap truc tiep vector theo ID.
- **pickle vecs**: Can cho `get_vectors_from_index()` (faiss_vs.py:38-41) de lay vectors theo indices. Duoc dung trong:
  - `sem_sim_join` khi load query embeddings tu index (sem_sim_join.py:114-117)
  - `FaissVS.__call__` khi search tren subset (faiss_vs.py:59)

---

## sem_index - Tao index (sem_index.py:9)

### SemIndexDataframe.__call__ (sem_index.py:62-77)

```python
@operator_cache
def __call__(self, col_name: str, index_dir: str) -> pd.DataFrame:
    rm = lotus.settings.rm
    vs = lotus.settings.vs
    ...
    embeddings = rm(self._obj[col_name].tolist())      # Tao embeddings
    vs.index(self._obj[col_name], embeddings, index_dir) # Luu index
    self._obj.attrs["index_dirs"][col_name] = index_dir  # Ghi nho index_dir
    return self._obj
```

Flow:
1. Lay RM va VS tu settings
2. Goi `rm(docs)` de tao embeddings cho tat ca documents
3. Goi `vs.index(docs, embeddings, index_dir)` de luu
4. Luu `index_dir` vao `attrs["index_dirs"][col_name]`

### Luu y: __init__ reset attrs (sem_index.py:54)

```python
def __init__(self, pandas_obj: Any) -> None:
    self._validate(pandas_obj)
    self._obj = pandas_obj
    self._obj.attrs["index_dirs"] = {}  # RESET moi lan!
```

**Canh bao**: Moi lan truy cap `df.sem_index`, `__init__` duoc goi va reset `attrs["index_dirs"]` thanh `{}`. Dieu nay co nghia chi co the dung `sem_index` 1 lan hoac phai chain calls.

---

## load_sem_index - Load index (load_sem_index.py:6)

### LoadSemIndexDataframe.__call__ (load_sem_index.py:49-51)

```python
def __call__(self, col_name: str, index_dir: str) -> pd.DataFrame:
    self._obj.attrs["index_dirs"][col_name] = index_dir
    return self._obj
```

**Quan trong**: `load_sem_index` CHI set metadata `attrs["index_dirs"]`. No KHONG thuc su load index vao memory. Index duoc load lazily khi operator can (vi du `sem_search` goi `vs.load_index()` tai sem_search.py:113).

### Tuong tu sem_index, __init__ reset attrs (load_sem_index.py:42):
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

- Kiem tra `vs.index_dir` co khop voi index can dung khong
- Chi load khi can thiet (index khac voi index hien tai)
- VS la singleton global (qua `lotus.settings.vs`), nen chi co 1 index duoc load tai 1 thoi diem

### Han che: Chi 1 index tai 1 thoi diem

Vi `lotus.settings.vs` la shared singleton, khi 2 operations can 2 index khac nhau, phai load/unload lien tuc. Vi du:

```python
df1.sem_search("col_a", "query1", K=5)  # Load col_a index
df2.sem_search("col_b", "query2", K=5)  # Load col_b index (unload col_a)
df1.sem_search("col_a", "query3", K=5)  # Load col_a index AGAIN
```

---

## ColBERTv2RM Persistence (colbertv2_rm.py:43-93)

ColBERTv2RM co persistence rieng, khong dung VS:

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

ColBERT luu index tai `experiments/lotus/indexes/` (hardcoded path).

---

## Index Propagation

Khi DataFrame duoc filter hoac transform, `index_dirs` attrs duoc propagate:

```python
# sem_filter.py:564
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)

# sem_search.py:141
new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)
```

Dieu nay cho phep chain operations: `df.sem_search(...).sem_filter(...)` ma van giu duoc index references.
