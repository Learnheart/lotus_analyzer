# LOTUS Class Inventory

> **lotus-ai v1.1.4** -- Danh sach day du tat ca cac class trong codebase.

## Core Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `Settings` | `lotus/settings.py:8` | -- | Cau hinh toan cuc: lm, rm, helper_lm, reranker, vs, enable_cache, serialization_format |

## Type System Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `LMOutput` | `lotus/types.py` | -- | Ket qua tra ve tu Language Model (outputs + logprobs) |
| `LMStats` | `lotus/types.py` | -- | Thong ke su dung LM (token count, cost) |
| `LogprobsForCascade` | `lotus/types.py` | -- | Log probabilities cho cascade optimization |
| `LogprobsForFilterCascade` | `lotus/types.py` | -- | Log probabilities danh rieng cho filter cascade |
| `SemanticMapPostprocessOutput` | `lotus/types.py` | -- | Ket qua hau xu ly cua sem_map |
| `SemanticMapOutput` | `lotus/types.py` | -- | Ket qua day du cua sem_map |
| `SemanticExtractPostprocessOutput` | `lotus/types.py` | -- | Ket qua hau xu ly cua sem_extract |
| `SemanticExtractOutput` | `lotus/types.py` | -- | Ket qua day du cua sem_extract |
| `SemanticFilterPostprocessOutput` | `lotus/types.py` | -- | Ket qua hau xu ly cua sem_filter |
| `SemanticFilterOutput` | `lotus/types.py` | -- | Ket qua day du cua sem_filter |
| `SemanticAggOutput` | `lotus/types.py` | -- | Ket qua cua sem_agg |
| `SemanticJoinOutput` | `lotus/types.py` | -- | Ket qua cua sem_join |
| `SemanticTopKOutput` | `lotus/types.py` | -- | Ket qua cua sem_topk |
| `RMOutput` | `lotus/types.py` | -- | Ket qua tra ve tu Retrieval Model |
| `RerankerOutput` | `lotus/types.py` | -- | Ket qua tra ve tu Reranker |
| `CascadeArgs` | `lotus/types.py` | -- | Tham so cau hinh cascade optimization |
| `UsageLimit` | `lotus/types.py` | -- | Gioi han su dung (token/cost) |
| `LotusException` | `lotus/types.py` | `Exception` | Exception co ban cua LOTUS |
| `LotusUsageLimitException` | `lotus/types.py` | `LotusException` | Exception khi vuot gioi han su dung |
| `LongContextStrategy` | `lotus/types.py` | `Enum` | Enum cho chien luoc xu ly van ban dai (TRUNCATE, CHUNK) |
| `ChunkInfo` | `lotus/types.py` | -- | Thong tin ve mot chunk van ban |
| `SerializationFormat` | `lotus/types.py` | `Enum` | Dinh dang serialize (JSON, XML, v.v.) |
| `ProxyModel` | `lotus/types.py` | -- | Cau hinh proxy model cho cascade |
| `ReasoningStrategy` | `lotus/types.py` | `Enum` | Chien luoc suy luan (CoT, v.v.) |

## Model Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `LM` | `lotus/models/lm.py:41` | -- | Language Model wrapper dua tren litellm; xu ly batching, caching, token counting |
| `RM` | `lotus/models/rm.py:10` | `ABC` | Abstract base class cho Retrieval Model |
| `Reranker` | `lotus/models/reranker.py:6` | `ABC` | Abstract base class cho Reranker |
| `SentenceTransformersRM` | `lotus/models/sentence_transformers_rm.py:11` | `RM` | Retrieval Model su dung sentence-transformers embeddings |
| `LiteLLMRM` | `lotus/models/litellm_rm.py:11` | `RM` | Retrieval Model su dung litellm embeddings API |
| `ColBERTv2RM` | `lotus/models/colbertv2_rm.py:17` | `RM` | Retrieval Model su dung ColBERTv2 server |
| `CrossEncoderReranker` | `lotus/models/cross_encoder_reranker.py:7` | `Reranker` | Reranker su dung cross-encoder model |

## Vector Store Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `VS` | `lotus/vector_store/vs.py:10` | `ABC` | Abstract base class cho Vector Store |
| `FaissVS` | `lotus/vector_store/faiss_vs.py:13` | `VS` | Vector store su dung FAISS (mac dinh, chay local) |
| `WeaviateVS` | `lotus/vector_store/weaviate_vs.py` | `VS` | Vector store su dung Weaviate (managed/self-hosted) |
| `QdrantVS` | `lotus/vector_store/qdrant_vs.py` | `VS` | Vector store su dung Qdrant |

## Cache Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `Cache` | `lotus/cache.py` | `ABC` | Abstract base class cho cache |
| `InMemoryCache` | `lotus/cache.py` | `Cache` | Cache trong bo nho (dictionary) |
| `SQLiteCache` | `lotus/cache.py` | `Cache` | Cache su dung SQLite database |
| `CacheFactory` | `lotus/cache.py` | -- | Factory tao cache instance theo cau hinh |
| `CacheConfig` | `lotus/cache.py` | -- | Cau hinh cache (type, path, v.v.) |
| `CacheType` | `lotus/cache.py` | `Enum` | Enum loai cache (MEMORY, SQLITE) |

## Dtype Extension Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `ImageDtype` | `lotus/dtype_extensions/image.py:12` | `pd.api.extensions.ExtensionDtype` | Custom pandas dtype cho hinh anh |
| `ImageArray` | `lotus/dtype_extensions/image.py:37` | `pd.api.extensions.ExtensionArray` | Custom pandas array chua du lieu hinh anh |

## Other Classes

| Name | File:Line | Parent | Mo ta |
|------|-----------|--------|-------|
| `ChunkedDocument` | `lotus/long_context_strategy.py` | -- | Van ban duoc chia thanh cac chunk |
| `WebSearchCorpus` | `lotus/web_search.py:14` | -- | Tap du lieu ket qua tim kiem web |

## DataFrame Accessor Classes

> Dang ky qua `@pd.api.extensions.register_dataframe_accessor("sem_xxx")`.
> Moi class wrap mot DataFrame va expose phuong thuc `__call__` la sem operator chinh.

| Name | File | Accessor Name | Mo ta |
|------|------|---------------|-------|
| `SemFilterDataframe` | `lotus/sem_ops/sem_filter.py` | `sem_filter` | Loc DataFrame theo ngu nghia |
| `SemMapDataframe` | `lotus/sem_ops/sem_map.py` | `sem_map` | Anh xa/chuyen doi cot bang LLM |
| `SemJoinDataframe` | `lotus/sem_ops/sem_join.py` | `sem_join` | Join ngu nghia 2 DataFrame |
| `SemAggDataframe` | `lotus/sem_ops/sem_agg.py` | `sem_agg` | Tong hop du lieu bang LLM |
| `SemTopKDataframe` | `lotus/sem_ops/sem_topk.py` | `sem_topk` | Chon top-K theo tieu chi ngu nghia |
| `SemExtractDataFrame` | `lotus/sem_ops/sem_extract.py` | `sem_extract` | Trich xuat thong tin tu cot |
| `SemSearchDataframe` | `lotus/sem_ops/sem_search.py` | `sem_search` | Tim kiem vector + reranking |
| `SemSimJoinDataframe` | `lotus/sem_ops/sem_sim_join.py` | `sem_sim_join` | Join theo do tuong dong vector |
| `SemDedupByDataframe` | `lotus/sem_ops/sem_dedup.py` | `sem_dedup` | Loai bo ban sao theo tuong dong |
| `SemIndexDataframe` | `lotus/sem_ops/sem_index.py` | `sem_index` | Tao vector index cho cot |
| `SemPartitionByDataframe` | `lotus/sem_ops/sem_partition_by.py` | `sem_partition_by` | Phan vung DataFrame |
| `SemClusterByDataframe` | `lotus/sem_ops/sem_cluster_by.py` | `sem_cluster_by` | Phan cum DataFrame theo vector |
| `LoadSemIndexDataframe` | `lotus/sem_ops/load_sem_index.py` | `load_sem_index` | Tai lai vector index da luu |
| `LLMAsJudgeDataframe` | `lotus/evals/llm_as_judge.py` | `llm_as_judge` | Danh gia bang LLM |
| `PairwiseJudgeDataframe` | `lotus/evals/pairwise_judge.py` | `pairwise_judge` | So sanh cap doi bang LLM |

## Thong ke tong hop

| Nhom | So luong class |
|------|----------------|
| Core / Settings | 1 |
| Type System | 21 |
| Models | 7 |
| Vector Stores | 4 |
| Cache | 5 |
| Dtype Extensions | 2 |
| Other | 2 |
| DataFrame Accessors | 15 |
| **Tong cong** | **57** |
