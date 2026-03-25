# Key Files Map — LOTUS

## Hướng dẫn đọc code theo chủ đề

| Tôi muốn hiểu... | Đọc file này | Bắt đầu từ dòng |
|-------------------|-------------|-----------------|
| **Kiến trúc tổng thể** | `lotus/__init__.py` | 1 — imports và __all__ cho thấy toàn bộ modules |
| **Cấu hình hệ thống** | `lotus/settings.py` | 5 — Settings class với tất cả cấu hình |
| **Data representation** | `lotus/dtype_extensions/image.py` | 12 — ImageDtype và ImageArray custom extension |
| **Cách DataFrame chứa metadata** | `lotus/sem_ops/sem_index.py` | 1 — attrs["index_dirs"] pattern |
| **LLM reasoning core** | `lotus/models/lm.py` | 41 — LM class: cache, batch, rate limit |
| **LLM call pipeline** | `lotus/models/lm.py` | 123 — `__call__` method: cache check → batch → stats |
| **Accuracy guarantees** | `lotus/sem_ops/cascade_utils.py` | 42 — `learn_cascade_thresholds` với Hoeffding bounds |
| **Statistical bounds** | `lotus/sem_ops/cascade_utils.py` | 52 — UB/LB functions |
| **Importance sampling** | `lotus/sem_ops/cascade_utils.py` | 8 — Weighted sampling với correction |
| **Structured ↔ Unstructured bridge** | `lotus/nl_expression.py` | 4 — parse_cols regex cho Langex |
| **Prompt construction** | `lotus/templates/task_instructions.py` | 1 — filter_formatter, map_formatter, etc. |
| **Filter operator** | `lotus/sem_ops/sem_filter.py` | 24 — Core logic: prompt → LM → postprocess |
| **Filter cascade** | `lotus/sem_ops/sem_filter.py` | 383 — Proxy routing + threshold learning |
| **Map operator** | `lotus/sem_ops/sem_map.py` | 14 — sem_map function |
| **Extract operator** | `lotus/sem_ops/sem_extract.py` | 15 — Structured extraction với JSON output |
| **Aggregation tree** | `lotus/sem_ops/sem_agg.py` | 60 — Hierarchical aggregation logic |
| **Top-K sorting** | `lotus/sem_ops/sem_topk.py` | 347 — llm_quicksort implementation |
| **Heapsort** | `lotus/sem_ops/sem_topk.py` | 560 — llm_heapsort với HeapDoc comparator |
| **TopK cascade** | `lotus/sem_ops/sem_topk.py` | 176 — compare_batch_binary_cascade |
| **Join core** | `lotus/sem_ops/sem_join.py` | 16 — sem_join: all-pairs filter |
| **Join cascade** | `lotus/sem_ops/sem_join.py` | 180 — sem_join_cascade orchestration |
| **Join optimizer** | `lotus/sem_ops/sem_join.py` | 417 — SF vs MSF comparison |
| **Join threshold learning** | `lotus/sem_ops/sem_join.py` | 530 — learn_join_cascade_threshold |
| **Semantic search** | `lotus/sem_ops/sem_search.py` | 10 — Vector search + post-filter |
| **Similarity join** | `lotus/sem_ops/sem_sim_join.py` | 12 — Embedding-based join |
| **Deduplication** | `lotus/sem_ops/sem_dedup.py` | 10 — Connected components dedup |
| **Clustering** | `lotus/sem_ops/sem_cluster_by.py` | 10 — KMeans clustering |
| **Partitioning** | `lotus/sem_ops/sem_partition_by.py` | 8 — Custom partition functions |
| **Post-processing** | `lotus/sem_ops/postprocessors.py` | 182 — filter_postprocess: parse True/False |
| **Extract post-processing** | `lotus/sem_ops/postprocessors.py` | 149 — JSON parsing, str casting |
| **CoT post-processing** | `lotus/sem_ops/postprocessors.py` | 12 — Chain-of-thought extraction |
| **Caching system** | `lotus/cache.py` | 33 — operator_cache decorator |
| **InMemoryCache** | `lotus/cache.py` | 247 — OrderedDict-based cache |
| **SQLiteCache** | `lotus/cache.py` | 168 — Persistent cache |
| **Type definitions** | `lotus/types.py` | 1 — LMOutput, CascadeArgs, etc. |
| **CascadeArgs** | `lotus/types.py` | 155 — Cascade configuration |
| **ProxyModel enum** | `lotus/types.py` | 150 — HELPER_LM vs EMBEDDING_MODEL |
| **Usage limits** | `lotus/types.py` | 215 — UsageLimit dataclass |
| **Retrieval model base** | `lotus/models/rm.py` | 10 — RM abstract class |
| **SentenceTransformers RM** | `lotus/models/sentence_transformers_rm.py` | 11 — Local embedding model |
| **LiteLLM RM** | `lotus/models/litellm_rm.py` | 11 — API-based embeddings |
| **ColBERTv2 RM** | `lotus/models/colbertv2_rm.py` | 17 — ColBERT retrieval |
| **Vector store base** | `lotus/vector_store/vs.py` | 1 — VS abstract class |
| **Faiss vector store** | `lotus/vector_store/faiss_vs.py` | 1 — FAISS implementation |
| **Long context handling** | `lotus/long_context_strategy.py` | 1 — ChunkedDocument + strategies |
| **Web search** | `lotus/web_search.py` | 1 — Web search integration |
| **Pricing** | `lotus/pricing.py` | 1 — Cost calculation |
| **Knowledge construction** | `lotus/data_connectors/connectors.py` | 1 — Data source connectors |

---

## Reading Order Recommendations

### Người mới bắt đầu — "LOTUS làm gì?"
1. `lotus/__init__.py:1` — Overview
2. `lotus/settings.py:1` — Configuration
3. `lotus/nl_expression.py:1` — Langex parsing
4. `lotus/sem_ops/sem_filter.py:225` — Operator pattern example
5. `lotus/models/lm.py:41` — LM class

### Engineer muốn contribute — "LOTUS hoạt động thế nào?"
1. `lotus/types.py:1` — Type system
2. `lotus/cache.py:33` — Caching pattern
3. `lotus/sem_ops/sem_filter.py:24` — Core operator logic
4. `lotus/models/lm.py:123` — LM call pipeline
5. `lotus/sem_ops/postprocessors.py:1` — Output parsing

### Researcher quan tâm optimization — "LOTUS tối ưu thế nào?"
1. `lotus/sem_ops/cascade_utils.py:42` — Threshold learning
2. `lotus/sem_ops/sem_filter.py:383` — Filter cascade
3. `lotus/sem_ops/sem_join.py:417` — Join optimizer
4. `lotus/sem_ops/sem_topk.py:176` — TopK cascade
5. `lotus/sem_ops/sem_agg.py:60` — Aggregation tree
