# A3 - Search Flow Trace: sem_search

## Trace: `df.sem_search("title", "AI", K=2)`

### Giả định setup:
```python
import lotus
from lotus.models import LM, SentenceTransformersRM
from lotus.vector_store import FaissVS

lotus.settings.configure(
    lm=LM(model="gpt-4o-mini"),
    rm=SentenceTransformersRM(model="intfloat/e5-base-v2"),
    vs=FaissVS()
)

df = pd.DataFrame({
    "title": ["Machine learning tutorial", "Data science guide", "Python basics"]
})
df = df.sem_index("title", "title_index")
# -> Tạo embeddings, lưu faiss index tại "title_index/"

df.sem_search("title", "AI", K=2)
```

---

### Step 1: SemSearchDataframe.__call__ (sem_search.py:92)

```
df.sem_search("title", "AI", K=2)
  |
  |-- pandas tạo SemSearchDataframe(df) -> self._obj = df
  |-- Gọi __call__("title", "AI", K=2)
  |-- @operator_cache kiểm tra cache (cache.py:33)
  |
  |-- K is not None -> vector search path
  |
  |-- Lấy rm và vs từ lotus.settings (sem_search.py:104-105):
  |     rm = lotus.settings.rm   # SentenceTransformersRM instance
  |     vs = lotus.settings.vs   # FaissVS instance
  |
  |-- Validate rm và vs không None (sem_search.py:106-109)
  |
  |-- Lấy index directory từ attrs (sem_search.py:111):
  |     col_index_dir = self._obj.attrs["index_dirs"]["title"]
  |     -> "title_index"
  |
  |-- Kiểm tra và load index nếu cần (sem_search.py:112-114):
  |     if vs.index_dir != col_index_dir:
  |         vs.load_index(col_index_dir)
  |     # Load faiss index + pickled vectors
```

### Step 2: Query Embedding (sem_search.py:121)

```
query_vectors = rm.convert_query_to_query_vector("AI")
  |
  |-- rm.convert_query_to_query_vector("AI")  (rm.py:53)
  |     |-- isinstance("AI", str) -> True
  |     |-- queries = ["AI"]  (rm.py:74)
  |     |-- query_vectors = rm._embed(["AI"])
  |           |-- SentenceTransformersRM._embed(["AI"])
  |           |     (sentence_transformers_rm.py:49)
  |           |-- convert_to_base_data(["AI"]) -> ["AI"]
  |           |-- transformer.encode(["AI"], convert_to_tensor=True,
  |           |     normalize_embeddings=True)
  |           |-- -> NDArray shape (1, 768)
  |
  |-- query_vectors: NDArray shape (1, 768)
```

### Step 3: Vector Search (sem_search.py:122)

```
vs_output: RMOutput = vs(query_vectors, search_K)
  |
  |-- FaissVS.__call__(query_vectors, K=2)  (faiss_vs.py:43)
  |     |-- ids is None -> search toàn bộ index
  |     |-- distances, indices = self.faiss_index.search(query_vectors, 2)
  |     |     (faiss_vs.py:75)
  |     |
  |     |-- Ví dụ kết quả:
  |     |     indices = [[0, 1]]     # global doc indices
  |     |     distances = [[0.85, 0.82]]
  |     |
  |     |-- Return RMOutput(distances=[[0.85, 0.82]], indices=[[0, 1]])
  |
  |-- doc_idxs = [0, 1]
  |-- scores = [0.85, 0.82]
```

### Step 4: Post-filter by df_idxs (sem_search.py:116-138)

```
df_idxs = self._obj.index  # [0, 1, 2]
  |
  |-- Iterative search loop (sem_search.py:120-138):
  |     while True:
  |         # Filter: chỉ giữ results có idx trong df_idxs
  |         postfiltered_doc_idxs = []
  |         postfiltered_scores = []
  |         for idx, score in zip(doc_idxs, scores):
  |             if idx in df_idxs:
  |                 postfiltered_doc_idxs.append(idx)
  |                 postfiltered_scores.append(score)
  |
  |         postfiltered_doc_idxs = postfiltered_doc_idxs[:K]
  |         postfiltered_scores = postfiltered_scores[:K]
  |
  |         if len(postfiltered_doc_idxs) == K:
  |             break
  |         search_K = search_K * 2  # Tăng search_K nếu không đủ kết quả
```

**Tại sao cần post-filter?**
Faiss index có thể chứa nhiều documents hơn DataFrame hiện tại (vì DataFrame có thể đã được filter trước đó). `df_idxs` là index của DataFrame hiện tại, chỉ giữ results có index thuộc DataFrame.

**Adaptive search**: Nếu post-filter loại bỏ quá nhiều kết quả (không đủ K), tăng `search_K *= 2` và search lại (sem_search.py:138).

### Step 5: Optional Reranking (sem_search.py:148-155)

```
if n_rerank is not None:
  |
  |-- reranker = lotus.settings.reranker
  |-- Validate reranker is not None (sem_search.py:149-150)
  |
  |-- docs = new_df[col_name].tolist()
  |     -> ["Machine learning tutorial", "Data science guide"]
  |
  |-- reranked_output = reranker(query, docs, n_rerank)
  |     (Reranker.__call__ -> cross-encoder scoring)
  |
  |-- reranked_idxs = reranked_output.indices
  |-- new_df = new_df.iloc[reranked_idxs]
```

Reranking chỉ chạy khi `n_rerank` được chỉ định. Có thể dùng kết hợp: `K=100, n_rerank=10` -> lấy 100 results bằng vector search, rồi rerank để chọn 10 tốt nhất.

### Step 6: Return Result (sem_search.py:140-157)

```
new_df = self._obj.loc[postfiltered_doc_idxs]
  |-- DataFrame({"title": ["Machine learning tutorial", "Data science guide"]})

new_df.attrs["index_dirs"] = self._obj.attrs.get("index_dirs", None)
  |-- Preserve index_dirs cho các operations tiếp theo

if return_scores:
    new_df["vec_scores_sim_score"] = postfiltered_scores
  |-- Thêm cột scores nếu được yêu cầu

Return new_df
```

## Tổng kết call stack

```
df.sem_search("title", "AI", K=2)
  -> SemSearchDataframe.__call__()             sem_search.py:92
    -> @operator_cache                         cache.py:33
    -> vs.load_index(index_dir)                faiss_vs.py:32
    -> rm.convert_query_to_query_vector("AI")  rm.py:53
      -> SentenceTransformersRM._embed()       sentence_transformers_rm.py:49
    -> vs(query_vectors, K)                    faiss_vs.py:43
      -> faiss_index.search()                  faiss (external)
    -> Post-filter by df_idxs                  sem_search.py:128-135
    -> [Optional] reranker(query, docs, n)     sem_search.py:148-155
    -> Return filtered DataFrame               sem_search.py:140-157
```
