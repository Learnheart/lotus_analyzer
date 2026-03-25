# E2 - Indexing Strategies

## Tổng quan

LOTUS sử dụng FAISS làm vector store chính. Indexing được trigger qua `sem_index` accessor và lưu trữ persistent trên disk.

---

## 1. sem_index Accessor

**Location**: `sem_ops/sem_index.py:9-77`

```python
@pd.api.extensions.register_dataframe_accessor("sem_index")
class SemIndexDataframe:
```

### Trigger indexing (`sem_index.py:62-77`)
```python
def __call__(self, col_name: str, index_dir: str) -> pd.DataFrame:
    rm = lotus.settings.rm          # :67
    vs = lotus.settings.vs          # :68
    embeddings = rm(self._obj[col_name].tolist())    # :74 - embed all values
    vs.index(self._obj[col_name], embeddings, index_dir)  # :75 - build index
    self._obj.attrs["index_dirs"][col_name] = index_dir   # :76 - save reference
    return self._obj                                       # :77
```

Process:
1. Lấy tất cả values từ column: `self._obj[col_name].tolist()`
2. Embed qua RM: `rm(docs)` → NDArray
3. Index qua VS: `vs.index(docs, embeddings, index_dir)`
4. Lưu index_dir reference trong DataFrame attrs

---

## 2. FaissVS - FAISS Vector Store

**Location**: `vector_store/faiss_vs.py:13-77`

### Configuration
```python
class FaissVS(VS):
    def __init__(self, factory_string="Flat", metric=faiss.METRIC_INNER_PRODUCT):  # :14
```

- `factory_string`: FAISS index type, default `"Flat"` (brute-force exact search)
- `metric`: default `faiss.METRIC_INNER_PRODUCT` - khi combined với normalized embeddings → cosine similarity

### Index creation (`faiss_vs.py:22-30`)
```python
def index(self, docs, embeddings, index_dir, **kwargs):
    self.faiss_index = faiss.index_factory(
        embeddings.shape[1], self.factory_string, self.metric    # :23
    )
    self.faiss_index.add(embeddings)                              # :24
    self.index_dir = index_dir                                    # :25
    os.makedirs(index_dir, exist_ok=True)                         # :27
    with open(f"{index_dir}/vecs", "wb") as fp:
        pickle.dump(embeddings, fp)                               # :29
    faiss.write_index(self.faiss_index, f"{index_dir}/index")     # :30
```

### Persistence format
Index directory chứa 2 files:
- `{index_dir}/index` - FAISS index binary (via `faiss.write_index`)
- `{index_dir}/vecs` - Raw embeddings pickle (via `pickle.dump`)

### Load index (`faiss_vs.py:32-36`)
```python
def load_index(self, index_dir):
    self.index_dir = index_dir                                     # :33
    self.faiss_index = faiss.read_index(f"{index_dir}/index")      # :34
    with open(f"{index_dir}/vecs", "rb") as fp:
        self.vecs = pickle.load(fp)                                # :36
```

---

## 3. Không có Incremental Indexing

**Quan trọng**: LOTUS không hỗ trợ incremental indexing.

- `vs.index()` luôn tạo index mới từ đầu (`faiss_vs.py:22-30`)
- Không có `add_to_index()` hay `update_index()` method
- Nếu data thay đổi, phải gọi `sem_index` lại từ đầu
- Toàn bộ embeddings được tính lại

---

## 4. load_sem_index

Để load index đã tạo trước đó:

```python
df.load_sem_index('title', 'title_index')
```

Hàm này chỉ set `attrs["index_dirs"]` trên DataFrame - không load vào memory ngay:
```python
self._obj.attrs["index_dirs"][col_name] = index_dir
```

FAISS index được load lazily khi `sem_search` cần:
```python
# sem_search.py:112-113
col_index_dir = self._obj.attrs["index_dirs"][col_name]
if vs.index_dir != col_index_dir:
    vs.load_index(col_index_dir)
```

---

## 5. Search operation

**Location**: `faiss_vs.py:43-77`

```python
def __call__(self, query_vectors, K, ids=None, **kwargs):
    if ids is not None:
        # Subset search - tạo temporary index
        subset_vecs = self.get_vectors_from_index(self.index_dir, ids)  # :59
        tmp_index = faiss.index_factory(...)                             # :63
        tmp_index.add(subset_vecs)                                       # :64
        distances, sub_indices = tmp_index.search(query_vectors, K)      # :67
        # Remap to original ids                                          # :71-72
    else:
        distances, indices = self.faiss_index.search(query_vectors, K)   # :75
```

Hỗ trợ:
- Full index search (default)
- Subset search: tạo temporary FAISS index cho subset of vectors

### get_vectors_from_index (`faiss_vs.py:38-41`)
```python
def get_vectors_from_index(self, index_dir, ids):
    with open(f"{index_dir}/vecs", "rb") as fp:
        vecs = pickle.load(fp)
    return vecs[ids]
```

Load embeddings từ disk và index bằng ids.

---

## 6. Factory String Options

FAISS `index_factory` hỗ trợ nhiều index types qua `factory_string`:
- `"Flat"` (default): Brute-force, exact results, O(N) search
- `"IVF100,Flat"`: Inverted file index, approximate, faster for large datasets
- `"HNSW32"`: Hierarchical Navigable Small World, very fast approximate search

Default `"Flat"` cho exact search - phù hợp cho small-medium datasets.

---

## 7. Kết luận

Indexing strategy của LOTUS đơn giản và effective:
- FAISS-based, configurable index types
- Persistent on disk (pickle + faiss binary)
- No incremental updates - rebuild required
- Lazy loading cho efficiency
- Default exact search, có thể scale với approximate indices
