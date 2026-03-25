# E3 - Semantic Similarity

## Tổng quan

LOTUS sử dụng dense vector similarity (FAISS) cho semantic retrieval. Default metric là inner product trên normalized vectors, tương đương cosine similarity.

---

## 1. Default Metric

**FaissVS**: `faiss.METRIC_INNER_PRODUCT` (`faiss_vs.py:14`)
```python
class FaissVS(VS):
    def __init__(self, factory_string="Flat", metric=faiss.METRIC_INNER_PRODUCT):
```

**SentenceTransformersRM**: `normalize_embeddings=True` by default (`sentence_transformers_rm.py:30`)
```python
def __init__(self, model="intfloat/e5-base-v2", max_batch_size=64,
             normalize_embeddings=True, device=None):
```

Khi embeddings được normalize (L2 norm = 1), inner product = cosine similarity:
```
cos(a, b) = (a · b) / (||a|| * ||b||) = a · b  (when ||a|| = ||b|| = 1)
```

---

## 2. Không có Explicit Threshold

LOTUS không sử dụng similarity threshold để quyết định match/no-match. Thay vào đó, dùng **K parameter** để control số kết quả:

- `sem_search`: `K` documents (`sem_search.py:96-97`)
- `sem_sim_join`: `K = len(l2)` (all documents) (`sem_join.py:360`)

Không có cách để user nói "chỉ trả về documents có similarity > 0.8". Luôn trả về top-K regardless of score.

---

## 3. sem_search vs sem_sim_join

### sem_search: Query → Documents

**Location**: `sem_search.py:92-157`

```python
query_vectors = rm.convert_query_to_query_vector(query)   # :121
vs_output = vs(query_vectors, search_K)                    # :122
```

- Input: 1 query string
- Output: top-K documents most similar to query
- Hỗ trợ post-filtering: chỉ giữ documents thuộc current DataFrame index (`sem_search.py:128-135`)
- Auto-expand search: nếu post-filtering loại quá nhiều, tăng `search_K *= 2` (`sem_search.py:138`)

### sem_sim_join: Documents → Documents

Dùng trong `run_sem_sim_join` (`sem_join.py:336-366`):
```python
l2_df = l2_df.sem_index(col2_label, f"{col2_label}_index")  # :358
out = l1_df.sem_sim_join(l2_df, left_on=col1_label, right_on=col2_label, K=K)  # :362
```

- Input: 2 sets of documents (l1, l2)
- Output: cho mỗi doc trong l1, tìm K nearest docs trong l2
- Dùng nội bộ cho join cascade optimization

---

## 4. Score Calibration

Trong join cascade, similarity scores được calibrate:

```python
# sem_join.py:365
out["_scores"] = calibrate_sem_sim_join(out["_scores"].tolist())
```

`calibrate_sem_sim_join` (`cascade_utils.py:147-149`):
```python
def calibrate_sem_sim_join(true_score):
    true_score = list(np.clip(true_score, 0, 1))
    return true_score
```

Chỉ clip scores vào range [0, 1] - không có complex calibration.

---

## 5. Reranking Integration

`sem_search` hỗ trợ optional reranking (`sem_search.py:148-155`):
```python
if n_rerank is not None:
    if lotus.settings.reranker is None:
        raise ValueError("Reranker not found in settings")
    docs = new_df[col_name].tolist()
    reranked_output = lotus.settings.reranker(query, docs, n_rerank)
    reranked_idxs = reranked_output.indices
    new_df = new_df.iloc[reranked_idxs]
```

Two-stage pipeline: vector search (K results) → reranker (n_rerank results).

---

## 6. Kết luận

- Pure dense vector similarity (no sparse/BM25)
- Cosine similarity via inner product + normalization
- K-based retrieval, no threshold
- Optional reranking for improved relevance
