# SEM_SEARCH — `sem_search`

## Metadata
- **File**: `lotus/sem_ops/sem_search.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_search")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_search.py:104-109)
- **LLM**: Không dùng LLM — pure vector search

## 1. Purpose & Use Cases

Tìm kiếm vector: query được convert thành embedding, so sánh với index đã xây. Hỗ trợ 2 giai đoạn: vector retrieval + optional reranking.

**Use cases:**
- Similarity search: `df.sem_search("title", "machine learning", K=5)`
- Reranked search: `df.sem_search("title", "AI", K=10, n_rerank=3)`
- Score analysis: `df.sem_search("title", "ML", K=5, return_scores=True)`

## 2. Call Stack Trace

```
1. SemSearchDataframe.__call__()                        # sem_search.py:92
2.   [Nếu K != None]:
2a.    rm = lotus.settings.rm                            # sem_search.py:104
2b.    vs = lotus.settings.vs                            # sem_search.py:105
2c.    Load col_index_dir từ df.attrs["index_dirs"]     # sem_search.py:111
2d.    [Nếu vs.index_dir != col_index_dir]:
2e.      vs.load_index(col_index_dir)                    # sem_search.py:113
2f.    Loop: rm.convert_query_to_query_vector(query)    # sem_search.py:121
2g.    vs(query_vectors, search_K)                       # sem_search.py:122
2h.    Post-filter: chỉ giữ indices trong df_idxs        # sem_search.py:128-133
2i.    [Nếu len(results) < K]: double search_K và retry  # sem_search.py:137-138
3.   [Nếu n_rerank != None]:
3a.    lotus.settings.reranker(query, docs, n_rerank)    # sem_search.py:153
3b.    Reindex DataFrame theo reranked order              # sem_search.py:155
4.   Return filtered DataFrame                           # sem_search.py:157
```

## 3. Prompt Template (COPY VERBATIM)

Không có prompt — sem_search không dùng LLM.

## 4. LLM Interaction

**Không có LLM interaction.** sem_search dùng:
- **Retrieval Model (RM)**: `rm.convert_query_to_query_vector(query)` (sem_search.py:121) — convert query string thành embedding vector
- **Vector Store (VS)**: `vs(query_vectors, search_K)` (sem_search.py:122) — KNN search trong index
- **Reranker** (optional): `lotus.settings.reranker(query, docs, n_rerank)` (sem_search.py:153) — cross-encoder reranking

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | N/A | Vector search là batch tự nhiên |
| Caching | Yes | `@operator_cache` (sem_search.py:91) |
| Cascading | N/A | |
| Early-termination | No | |
| Post-filtering | Yes | Filter kết quả theo DataFrame index (sem_search.py:128-133) |
| Adaptive K | Yes | Double search_K khi không đủ results (sem_search.py:137-138) |

### Adaptive search K chi tiết:
```python
search_K = K
while True:
    vs_output = vs(query_vectors, search_K)
    # Post-filter...
    if len(postfiltered_doc_idxs) == K:
        break
    search_K = search_K * 2  # sem_search.py:138
```
Logic này đảm bảo luôn trả về đúng K results, kể cả khi index có nhiều items đã bị remove khỏi DataFrame.

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column đã được index (sem_search.py:94)
- `query: str` — Natural language query (sem_search.py:95)
- `K: int | None` — Số documents trả về từ vector search (sem_search.py:96)
- `n_rerank: int | None` — Số documents trả về sau reranking (sem_search.py:97)
- `return_scores: bool` — Trả về similarity scores (sem_search.py:98)
- `suffix: str` — Suffix cho score column (sem_search.py:99, default="_sim_score")

### Output:
- DataFrame với K (hoặc n_rerank) rows, sorted by similarity (sem_search.py:140-141)
- **return_scores=True**: Thêm column `vec_scores{suffix}` (sem_search.py:144)
- **index_dirs preserved**: `new_df.attrs["index_dirs"]` được copy từ original (sem_search.py:141)

### Yêu cầu:
- Column phải được index trước bằng `sem_index()` (sem_search.py:111)
- `K` hoặc `n_rerank` phải != None (sem_search.py:101)

## 7. Edge Cases

1. **K và n_rerank đều None**: Assert error (sem_search.py:101)
2. **Column chưa được index**: KeyError khi truy cập `df.attrs["index_dirs"][col_name]` (sem_search.py:111)
3. **K > len(df)**: Clamp xuống `len(df_idxs)` (sem_search.py:118)
4. **Post-filter loại hết**: Loop vô hạn — search_K double mãi (sem_search.py:120-138). Tuy nhiên bị giới hạn bởi `cur_min` cap (sem_search.py:117-118)
5. **Reranker chưa set**: Raise `ValueError` (sem_search.py:149-150)
6. **Index chưa load**: Tự động load (sem_search.py:112-113)

## 8. Code Examples

```python
# Setup
df = df.sem_index("title", "title_index")

# Basic search
df.sem_search("title", "machine learning", K=5)

# Với scores
df.sem_search("title", "AI", K=10, return_scores=True)

# Với reranking
df.sem_search("title", "deep learning", K=20, n_rerank=5)

# Load index từ disk
df.load_sem_index("title", "title_index")
df.sem_search("title", "NLP", K=3)
```

## 9. Assessment

### Điểm mạnh:
- **No LLM cost**: Pure vector search — nhanh và rẻ
- **Two-stage pipeline**: Vector retrieval + reranking cho kết quả tốt hơn
- **Adaptive K**: Tự động tăng search range khi post-filter loại nhiều
- **Index preservation**: `index_dirs` attrs được truyền cho downstream operators

### Điểm yếu:
- **Potential infinite loop**: search_K double loop không có explicit break condition ngoài K satisfaction (sem_search.py:120-138)
- **Single query only**: Không hỗ trợ batch queries
- **No score normalization**: Scores là raw distances từ vector store
- **Reranker separate from RM**: Phải configure riêng `lotus.settings.reranker`
