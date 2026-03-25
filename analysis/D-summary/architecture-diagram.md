# Architecture Diagram — LOTUS

## Tổng thể: Sơ đồ ASCII toàn bộ kiến trúc

```
+==============================================================================+
|                          USER APPLICATION LAYER                               |
|                                                                               |
|   import lotus                                                                |
|   lotus.settings.configure(lm=LM(...), rm=RM(...), vs=VS(...))              |
|   df.sem_filter("...").sem_map("...").sem_topk("...", K=5)                   |
+==============================================================================+
        |                          |                           |
        | Langex parsing           | Settings access           | DataFrame I/O
        | nl_expression.py:4-8     | settings.py:5-35          |
        v                          v                           v
+==============================================================================+
|                        SEMANTIC OPERATOR LAYER                                |
|                     (Pandas DataFrame Accessors)                              |
|                                                                               |
|  +-------------+  +----------+  +-----------+  +----------+  +------------+  |
|  | sem_filter   |  | sem_map  |  | sem_extract|  | sem_agg  |  | sem_topk   |  |
|  | sem_filter.py|  |sem_map.py|  |sem_extract |  |sem_agg.py|  |sem_topk.py |  |
|  |  :225       |  |  :121    |  |  .py:111   |  |  :226    |  |  :624      |  |
|  +------+------+  +----+-----+  +-----+-----+  +----+-----+  +-----+------+  |
|         |              |              |              |              |          |
|  +-------------+  +----------+  +-----------+  +----------+  +------------+  |
|  | sem_join     |  |sem_search|  |sem_sim_join|  |sem_dedup |  |sem_cluster |  |
|  | sem_join.py  |  |sem_search|  |sem_sim_join|  |sem_dedup |  |sem_cluster |  |
|  |  :606       |  |  .py:10  |  |  .py:12    |  |  .py:10  |  | _by.py:10  |  |
|  +------+------+  +----+-----+  +-----+-----+  +----+-----+  +-----+------+  |
|         |              |              |              |              |          |
|  +-------------------+  +-----------------------+  +-------------------+      |
|  | sem_partition_by   |  | sem_index / load_index|  | web_search        |      |
|  | sem_partition_by.py|  | sem_index.py          |  | web_search.py     |      |
|  |  :8               |  | load_sem_index.py     |  |                   |      |
|  +-------------------+  +-----------------------+  +-------------------+      |
+==============================================================================+
        |                    |                    |                    |
        | @operator_cache    | Cascade routing    | Prompt templates   | Postprocess
        | cache.py:33        | cascade_utils.py   | task_instructions  | postprocessors
        v                    v                    v                    v
+==============================================================================+
|                       OPTIMIZATION LAYER                                      |
|                                                                               |
|  +------------------+  +--------------------+  +------------------------+     |
|  | Operator Cache   |  | Model Cascading    |  | Join Optimizer         |     |
|  | cache.py:33-100  |  | cascade_utils.py   |  | sem_join.py:417-527    |     |
|  | SHA256(df+args)  |  | :42-144            |  | SF vs MSF comparison   |     |
|  +------------------+  +--------------------+  +------------------------+     |
|                                                                               |
|  +------------------+  +--------------------+  +------------------------+     |
|  | safe_mode        |  | Usage Limits       |  | Embedding Pre-filter   |     |
|  | Cost estimation  |  | types.py:215-220   |  | sem_topk.py:782-788    |     |
|  +------------------+  +--------------------+  +------------------------+     |
+==============================================================================+
        |                              |                              |
        v                              v                              v
+==============================================================================+
|                         MODEL LAYER                                           |
|                                                                               |
|  +---------------------------+      +---------------------------+             |
|  |    LM (Language Model)    |      |    RM (Retrieval Model)   |             |
|  |    models/lm.py:41        |      |    models/rm.py:10        |             |
|  |                           |      |                           |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |  | Response Cache      |  |      |  | SentenceTransformers |  |             |
|  |  | SHA256(model+msg)   |  |      |  | RM  :11              |  |             |
|  |  | lm.py:407-410       |  |      |  | sentence_transformers|  |             |
|  |  +---------------------+  |      |  | _rm.py               |  |             |
|  |                           |      |  +---------------------+  |             |
|  |  +---------------------+  |      |                           |             |
|  |  | Batch Processing    |  |      |  +---------------------+  |             |
|  |  | batch_completion    |  |      |  | LiteLLMRM  :11       |  |             |
|  |  | lm.py:250-252       |  |      |  | litellm_rm.py        |  |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |                           |      |                           |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |  | Rate/TPM Limiting   |  |      |  | ColBERTv2RM          |  |             |
|  |  | lm.py:258-390       |  |      |  | colbertv2_rm.py:17   |  |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |                           |      |                           |             |
|  |  +---------------------+  |      +---------------------------+             |
|  |  | Usage Stats         |  |                                                |
|  |  | types.py:20-66      |  |      +---------------------------+             |
|  |  | virtual + physical  |  |      |    Reranker               |             |
|  |  +---------------------+  |      |    models/reranker.py     |             |
|  +---------------------------+      |    cross_encoder_reranker  |             |
|                                     +---------------------------+             |
+==============================================================================+
        |                              |                              |
        v                              v                              v
+==============================================================================+
|                       STORAGE / BACKEND LAYER                                 |
|                                                                               |
|  +---------------------------+      +---------------------------+             |
|  |    Cache Backends         |      |    Vector Store (VS)      |             |
|  |    cache.py               |      |    vector_store/vs.py     |             |
|  |                           |      |                           |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |  | InMemoryCache :247  |  |      |  | FaissVS             |  |             |
|  |  | OrderedDict LRU     |  |      |  | faiss_vs.py         |  |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |                           |      |                           |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  |  | SQLiteCache :168    |  |      |  | QdrantVS            |  |             |
|  |  | ~/.lotus/cache/     |  |      |  | qdrant_vs.py        |  |             |
|  |  +---------------------+  |      |  +---------------------+  |             |
|  +---------------------------+      |                           |             |
|                                     |  +---------------------+  |             |
|                                     |  | WeaviateVS          |  |             |
|                                     |  | weaviate_vs.py      |  |             |
|                                     |  +---------------------+  |             |
|                                     +---------------------------+             |
+==============================================================================+
        |                              |
        v                              v
+==============================================================================+
|                      EXTERNAL SERVICES                                        |
|                                                                               |
|  +---------------------------+      +---------------------------+             |
|  |  LiteLLM (LLM Gateway)   |      |  Embedding APIs           |             |
|  |  OpenAI, Anthropic,      |      |  OpenAI, Cohere,          |             |
|  |  Ollama, DeepSeek, etc.   |      |  HuggingFace, etc.        |             |
|  +---------------------------+      +---------------------------+             |
+==============================================================================+
```

---

## Data Flow: Một query điển hình

```
df.sem_filter("Is {text} positive?")
  |
  v
[1] nl_expression.parse_cols("Is {text} positive?")  →  ["text"]
  |
  v
[2] task_instructions.df2multimodal_info(df, ["text"])
  |   Chuyển mỗi row thành dict: {"text": "...", "images": [...]}
  v
[3] task_instructions.filter_formatter(model, doc, instruction, ...)
  |   Tạo prompt cho mỗi row
  v
[4] model(inputs)  →  LM.__call__(messages)
  |   |
  |   +-- Check cache (lm.py:136-149)
  |   +-- batch_completion (lm.py:250)
  |   +-- Update stats (lm.py:168-177)
  |   +-- Return LMOutput
  v
[5] filter_postprocess(outputs)  →  [True, False, True, ...]
  |   Parse "True"/"False" từ LLM output (postprocessors.py:182-218)
  v
[6] Return filtered DataFrame
```

---

## Data Flow: Model Cascading

```
df.sem_filter("Is {text} positive?", cascade_args=CascadeArgs(...))
  |
  v
[1] Run proxy model on ALL data
  |   Helper LM (logprobs) OR Embedding similarity
  v
[2] calibrate_llm_logprobs() → proxy_scores ∈ [0, 1]
  |
  v
[3] importance_sampling() → sample_indices, correction_factors
  |
  v
[4] Run oracle LM on SAMPLE only → oracle_outputs
  |
  v
[5] learn_cascade_thresholds() → (tau_pos, tau_neg)
  |
  v
[6] Route:
    +-- score >= tau_pos → Accept (proxy says True)
    +-- score <= tau_neg → Reject (proxy says False)
    +-- otherwise → Send to oracle LM
  |
  v
[7] Return filtered DataFrame
```
