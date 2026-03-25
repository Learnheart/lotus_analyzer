# E3 - Hybrid Search

## Tổng quan

LOTUS **KHÔNG** implement hybrid search. Hệ thống chỉ hỗ trợ **purely dense vector search** - không có BM25, TF-IDF, hay bất kỳ sparse retrieval method nào.

---

## 1. Chỉ có Dense Vector Search

Toàn bộ retrieval pipeline:

```
Text → RM._embed() → Dense Vector → FAISS Index → KNN Search
```

Các components:
- **Embedding**: `SentenceTransformersRM._embed()` (`sentence_transformers_rm.py:49-76`) hoặc `LiteLLMRM._embed()` (`litellm_rm.py:45-71`)
- **Indexing**: `FaissVS.index()` (`faiss_vs.py:22-30`)
- **Search**: `FaissVS.__call__()` (`faiss_vs.py:43-77`)

---

## 2. Không có Sparse Retrieval

Không tìm thấy trong codebase:
- BM25 implementation
- TF-IDF computation
- Inverted index (cho text matching)
- Keyword matching
- Any form of lexical search

---

## 3. Không có Hybrid Combination

Không có:
- Score fusion (RRF, linear combination)
- Multi-stage retrieval (sparse → dense)
- Ensemble methods

---

## 4. Single Retrieval Mode

Chỉ có **MỘT** retrieval mode:

```python
# sem_search.py:121-124
query_vectors = rm.convert_query_to_query_vector(query)
vs_output = vs(query_vectors, search_K)
doc_idxs = vs_output.indices[0]
scores = vs_output.distances[0]
```

1. Convert query thành embedding vector
2. KNN search trên FAISS index
3. Trả về top-K documents

---

## 5. Implications

- **Pro**: Đơn giản, consistent behavior
- **Pro**: Semantic understanding qua dense embeddings
- **Con**: Miss exact keyword matches mà dense embedding có thể bỏ lỡ
- **Con**: Không tối ưu cho queries cần exact term matching (tên người, mã sản phẩm, etc.)
- **Con**: Không có fallback mechanism khi dense search fail

---

## 6. Kết luận

LOTUS chọn simplicity over completeness cho retrieval. Chỉ dense vector search. Nếu cần hybrid search, user phải implement bên ngoài LOTUS.
