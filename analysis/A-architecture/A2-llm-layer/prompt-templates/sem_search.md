# Prompt Template: sem_search

## Khong co LLM Prompt

`sem_search` la purely vector-based retrieval operator. No KHONG su dung LLM va KHONG co prompt template.

## Flow

```
sem_search(col_name, query, K)
    |
    |-- rm.convert_query_to_query_vector(query)   # Embedding model
    |-- vs(query_vectors, K)                       # Vector store search
    |-- Post-filter by df_idxs                     # Loc theo DataFrame index
    |-- [Optional] reranker(query, docs, n_rerank) # Cross-encoder reranking
```

Xem chi tiet tai `/analysis/A-architecture/A3-embedding-layer/search-flow-trace.md`.

## Optional Reranking

Khi `n_rerank` duoc chi dinh (sem_search.py:148-155), su dung `lotus.settings.reranker`:

```python
if n_rerank is not None:
    if lotus.settings.reranker is None:
        raise ValueError("Reranker not found in settings")
    docs = new_df[col_name].tolist()
    reranked_output: RerankerOutput = lotus.settings.reranker(query, docs, n_rerank)
    reranked_idxs = reranked_output.indices
    new_df = new_df.iloc[reranked_idxs]
```

Reranker (vi du CrossEncoderReranker tai `models/cross_encoder_reranker.py`) co the dung cross-encoder model de re-score documents, nhung van khong phai LLM prompt.

## So sanh voi cac operators khac

| Aspect | sem_search | sem_filter | sem_topk |
|---|---|---|---|
| Model | RM (embedding) | LM (language) | LM (language) |
| Prompt | None | filter_formatter | get_match_prompt_binary |
| Ranking | Vector similarity | Boolean (pass/fail) | Pairwise comparison |
| Cost | Low (embedding only) | High (LLM per row) | Very high (LLM per pair) |
| Quality | Approximate | Exact (LLM judgment) | Exact (LLM judgment) |
