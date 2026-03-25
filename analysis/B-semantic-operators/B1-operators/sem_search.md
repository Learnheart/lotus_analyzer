# SEM_SEARCH — `sem_search`

## Metadata
- **File**: `lotus/sem_ops/sem_search.py`
- **Accessor line**: 10 (`@pd.api.extensions.register_dataframe_accessor("sem_search")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.rm` + `lotus.settings.vs` (sem_search.py:104-109)
- **LLM**: Khong dung LLM — pure vector search

## 1. Purpose & Use Cases

Tim kiem vector: query duoc convert thanh embedding, so sanh voi index da xay. Ho tro 2 giai doan: vector retrieval + optional reranking.

**Use cases:**
- Similarity search: `df.sem_search("title", "machine learning", K=5)`
- Reranked search: `df.sem_search("title", "AI", K=10, n_rerank=3)`
- Score analysis: `df.sem_search("title", "ML", K=5, return_scores=True)`

## 2. Call Stack Trace

```
1. SemSearchDataframe.__call__()                        # sem_search.py:92
2.   [Neu K != None]:
2a.    rm = lotus.settings.rm                            # sem_search.py:104
2b.    vs = lotus.settings.vs                            # sem_search.py:105
2c.    Load col_index_dir tu df.attrs["index_dirs"]     # sem_search.py:111
2d.    [Neu vs.index_dir != col_index_dir]:
2e.      vs.load_index(col_index_dir)                    # sem_search.py:113
2f.    Loop: rm.convert_query_to_query_vector(query)    # sem_search.py:121
2g.    vs(query_vectors, search_K)                       # sem_search.py:122
2h.    Post-filter: chi giu indices trong df_idxs        # sem_search.py:128-133
2i.    [Neu len(results) < K]: double search_K va retry  # sem_search.py:137-138
3.   [Neu n_rerank != None]:
3a.    lotus.settings.reranker(query, docs, n_rerank)    # sem_search.py:153
3b.    Reindex DataFrame theo reranked order              # sem_search.py:155
4.   Return filtered DataFrame                           # sem_search.py:157
```

## 3. Prompt Template (COPY VERBATIM)

Khong co prompt — sem_search khong dung LLM.

## 4. LLM Interaction

**Khong co LLM interaction.** sem_search dung:
- **Retrieval Model (RM)**: `rm.convert_query_to_query_vector(query)` (sem_search.py:121) — convert query string thanh embedding vector
- **Vector Store (VS)**: `vs(query_vectors, search_K)` (sem_search.py:122) — KNN search trong index
- **Reranker** (optional): `lotus.settings.reranker(query, docs, n_rerank)` (sem_search.py:153) — cross-encoder reranking

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | N/A | Vector search la batch tu nhien |
| Caching | Yes | `@operator_cache` (sem_search.py:91) |
| Cascading | N/A | |
| Early-termination | No | |
| Post-filtering | Yes | Filter ket qua theo DataFrame index (sem_search.py:128-133) |
| Adaptive K | Yes | Double search_K khi khong du results (sem_search.py:137-138) |

### Adaptive search K chi tiet:
```python
search_K = K
while True:
    vs_output = vs(query_vectors, search_K)
    # Post-filter...
    if len(postfiltered_doc_idxs) == K:
        break
    search_K = search_K * 2  # sem_search.py:138
```
Logic nay dam bao luon tra ve dung K results, ke ca khi index co nhieu items da bi remove khoi DataFrame.

## 6. Input/Output Contract

### Input:
- `col_name: str` — Column da duoc index (sem_search.py:94)
- `query: str` — Natural language query (sem_search.py:95)
- `K: int | None` — So documents tra ve tu vector search (sem_search.py:96)
- `n_rerank: int | None` — So documents tra ve sau reranking (sem_search.py:97)
- `return_scores: bool` — Tra ve similarity scores (sem_search.py:98)
- `suffix: str` — Suffix cho score column (sem_search.py:99, default="_sim_score")

### Output:
- DataFrame voi K (hoac n_rerank) rows, sorted by similarity (sem_search.py:140-141)
- **return_scores=True**: Them column `vec_scores{suffix}` (sem_search.py:144)
- **index_dirs preserved**: `new_df.attrs["index_dirs"]` duoc copy tu original (sem_search.py:141)

### Yeu cau:
- Column phai duoc index truoc bang `sem_index()` (sem_search.py:111)
- `K` hoac `n_rerank` phai != None (sem_search.py:101)

## 7. Edge Cases

1. **K va n_rerank deu None**: Assert error (sem_search.py:101)
2. **Column chua duoc index**: KeyError khi truy cap `df.attrs["index_dirs"][col_name]` (sem_search.py:111)
3. **K > len(df)**: Clamp xuong `len(df_idxs)` (sem_search.py:118)
4. **Post-filter loai het**: Loop vo han — search_K double mai (sem_search.py:120-138). Tuy nhien bi gioi han boi `cur_min` cap (sem_search.py:117-118)
5. **Reranker chua set**: Raise `ValueError` (sem_search.py:149-150)
6. **Index chua load**: Tu dong load (sem_search.py:112-113)

## 8. Code Examples

```python
# Setup
df = df.sem_index("title", "title_index")

# Basic search
df.sem_search("title", "machine learning", K=5)

# Voi scores
df.sem_search("title", "AI", K=10, return_scores=True)

# Voi reranking
df.sem_search("title", "deep learning", K=20, n_rerank=5)

# Load index tu disk
df.load_sem_index("title", "title_index")
df.sem_search("title", "NLP", K=3)
```

## 9. Assessment

### Diem manh:
- **No LLM cost**: Pure vector search — nhanh va re
- **Two-stage pipeline**: Vector retrieval + reranking cho ket qua tot hon
- **Adaptive K**: Tu dong tang search range khi post-filter loai nhieu
- **Index preservation**: `index_dirs` attrs duoc truyen cho downstream operators

### Diem yeu:
- **Potential infinite loop**: search_K double loop khong co explicit break condition ngoai K satisfaction (sem_search.py:120-138)
- **Single query only**: Khong ho tro batch queries
- **No score normalization**: Scores la raw distances tu vector store
- **Reranker separate from RM**: Phai configure rieng `lotus.settings.reranker`
