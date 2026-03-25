# LOTUS Class Inventory

> **lotus-ai v1.1.4** -- Danh sách đầy đủ tất cả các class trong codebase.

## Core Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `Settings` | `lotus/settings.py:8` | -- | Cấu hình toàn cục: lm, rm, helper_lm, reranker, vs, enable_cache, serialization_format |

## Type System Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `LMOutput` | `lotus/types.py` | -- | Kết quả trả về từ Language Model (outputs + logprobs) |
| `LMStats` | `lotus/types.py` | -- | Thống kê sử dụng LM (token count, cost) |
| `LogprobsForCascade` | `lotus/types.py` | -- | Log probabilities cho cascade optimization |
| `LogprobsForFilterCascade` | `lotus/types.py` | -- | Log probabilities dành riêng cho filter cascade |
| `SemanticMapPostprocessOutput` | `lotus/types.py` | -- | Kết quả hậu xử lý của sem_map |
| `SemanticMapOutput` | `lotus/types.py` | -- | Kết quả đầy đủ của sem_map |
| `SemanticExtractPostprocessOutput` | `lotus/types.py` | -- | Kết quả hậu xử lý của sem_extract |
| `SemanticExtractOutput` | `lotus/types.py` | -- | Kết quả đầy đủ của sem_extract |
| `SemanticFilterPostprocessOutput` | `lotus/types.py` | -- | Kết quả hậu xử lý của sem_filter |
| `SemanticFilterOutput` | `lotus/types.py` | -- | Kết quả đầy đủ của sem_filter |
| `SemanticAggOutput` | `lotus/types.py` | -- | Kết quả của sem_agg |
| `SemanticJoinOutput` | `lotus/types.py` | -- | Kết quả của sem_join |
| `SemanticTopKOutput` | `lotus/types.py` | -- | Kết quả của sem_topk |
| `RMOutput` | `lotus/types.py` | -- | Kết quả trả về từ Retrieval Model |
| `RerankerOutput` | `lotus/types.py` | -- | Kết quả trả về từ Reranker |
| `CascadeArgs` | `lotus/types.py` | -- | Tham số cấu hình cascade optimization |
| `UsageLimit` | `lotus/types.py` | -- | Giới hạn sử dụng (token/cost) |
| `LotusException` | `lotus/types.py` | `Exception` | Exception cơ bản của LOTUS |
| `LotusUsageLimitException` | `lotus/types.py` | `LotusException` | Exception khi vượt giới hạn sử dụng |
| `LongContextStrategy` | `lotus/types.py` | `Enum` | Enum cho chiến lược xử lý văn bản dài (TRUNCATE, CHUNK) |
| `ChunkInfo` | `lotus/types.py` | -- | Thông tin về một chunk văn bản |
| `SerializationFormat` | `lotus/types.py` | `Enum` | Định dạng serialize (JSON, XML, v.v.) |
| `ProxyModel` | `lotus/types.py` | -- | Cấu hình proxy model cho cascade |
| `ReasoningStrategy` | `lotus/types.py` | `Enum` | Chiến lược suy luận (CoT, v.v.) |

## Model Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `LM` | `lotus/models/lm.py:41` | -- | Language Model wrapper dựa trên litellm; xử lý batching, caching, token counting |
| `RM` | `lotus/models/rm.py:10` | `ABC` | Abstract base class cho Retrieval Model |
| `Reranker` | `lotus/models/reranker.py:6` | `ABC` | Abstract base class cho Reranker |
| `SentenceTransformersRM` | `lotus/models/sentence_transformers_rm.py:11` | `RM` | Retrieval Model sử dụng sentence-transformers embeddings |
| `LiteLLMRM` | `lotus/models/litellm_rm.py:11` | `RM` | Retrieval Model sử dụng litellm embeddings API |
| `ColBERTv2RM` | `lotus/models/colbertv2_rm.py:17` | `RM` | Retrieval Model sử dụng ColBERTv2 server |
| `CrossEncoderReranker` | `lotus/models/cross_encoder_reranker.py:7` | `Reranker` | Reranker sử dụng cross-encoder model |

## Vector Store Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `VS` | `lotus/vector_store/vs.py:10` | `ABC` | Abstract base class cho Vector Store |
| `FaissVS` | `lotus/vector_store/faiss_vs.py:13` | `VS` | Vector store sử dụng FAISS (mặc định, chạy local) |
| `WeaviateVS` | `lotus/vector_store/weaviate_vs.py` | `VS` | Vector store sử dụng Weaviate (managed/self-hosted) |
| `QdrantVS` | `lotus/vector_store/qdrant_vs.py` | `VS` | Vector store sử dụng Qdrant |

## Cache Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `Cache` | `lotus/cache.py` | `ABC` | Abstract base class cho cache |
| `InMemoryCache` | `lotus/cache.py` | `Cache` | Cache trong bộ nhớ (dictionary) |
| `SQLiteCache` | `lotus/cache.py` | `Cache` | Cache sử dụng SQLite database |
| `CacheFactory` | `lotus/cache.py` | -- | Factory tạo cache instance theo cấu hình |
| `CacheConfig` | `lotus/cache.py` | -- | Cấu hình cache (type, path, v.v.) |
| `CacheType` | `lotus/cache.py` | `Enum` | Enum loại cache (MEMORY, SQLITE) |

## Dtype Extension Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `ImageDtype` | `lotus/dtype_extensions/image.py:12` | `pd.api.extensions.ExtensionDtype` | Custom pandas dtype cho hình ảnh |
| `ImageArray` | `lotus/dtype_extensions/image.py:37` | `pd.api.extensions.ExtensionArray` | Custom pandas array chứa dữ liệu hình ảnh |

## Other Classes

| Name | File:Line | Parent | Mô tả |
|------|-----------|--------|-------|
| `ChunkedDocument` | `lotus/long_context_strategy.py` | -- | Văn bản được chia thành các chunk |
| `WebSearchCorpus` | `lotus/web_search.py:14` | -- | Tập dữ liệu kết quả tìm kiếm web |

## DataFrame Accessor Classes

> Đăng ký qua `@pd.api.extensions.register_dataframe_accessor("sem_xxx")`.
> Mỗi class wrap một DataFrame và expose phương thức `__call__` là sem operator chính.

| Name | File | Accessor Name | Mô tả |
|------|------|---------------|-------|
| `SemFilterDataframe` | `lotus/sem_ops/sem_filter.py` | `sem_filter` | Lọc DataFrame theo ngữ nghĩa |
| `SemMapDataframe` | `lotus/sem_ops/sem_map.py` | `sem_map` | Ánh xạ/chuyển đổi cột bằng LLM |
| `SemJoinDataframe` | `lotus/sem_ops/sem_join.py` | `sem_join` | Join ngữ nghĩa 2 DataFrame |
| `SemAggDataframe` | `lotus/sem_ops/sem_agg.py` | `sem_agg` | Tổng hợp dữ liệu bằng LLM |
| `SemTopKDataframe` | `lotus/sem_ops/sem_topk.py` | `sem_topk` | Chọn top-K theo tiêu chí ngữ nghĩa |
| `SemExtractDataFrame` | `lotus/sem_ops/sem_extract.py` | `sem_extract` | Trích xuất thông tin từ cột |
| `SemSearchDataframe` | `lotus/sem_ops/sem_search.py` | `sem_search` | Tìm kiếm vector + reranking |
| `SemSimJoinDataframe` | `lotus/sem_ops/sem_sim_join.py` | `sem_sim_join` | Join theo độ tương đồng vector |
| `SemDedupByDataframe` | `lotus/sem_ops/sem_dedup.py` | `sem_dedup` | Loại bỏ bản sao theo tương đồng |
| `SemIndexDataframe` | `lotus/sem_ops/sem_index.py` | `sem_index` | Tạo vector index cho cột |
| `SemPartitionByDataframe` | `lotus/sem_ops/sem_partition_by.py` | `sem_partition_by` | Phân vùng DataFrame |
| `SemClusterByDataframe` | `lotus/sem_ops/sem_cluster_by.py` | `sem_cluster_by` | Phân cụm DataFrame theo vector |
| `LoadSemIndexDataframe` | `lotus/sem_ops/load_sem_index.py` | `load_sem_index` | Tải lại vector index đã lưu |
| `LLMAsJudgeDataframe` | `lotus/evals/llm_as_judge.py` | `llm_as_judge` | Đánh giá bằng LLM |
| `PairwiseJudgeDataframe` | `lotus/evals/pairwise_judge.py` | `pairwise_judge` | So sánh cặp đôi bằng LLM |

## Thống kê tổng hợp

| Nhóm | Số lượng class |
|------|----------------|
| Core / Settings | 1 |
| Type System | 21 |
| Models | 7 |
| Vector Stores | 4 |
| Cache | 5 |
| Dtype Extensions | 2 |
| Other | 2 |
| DataFrame Accessors | 15 |
| **Tổng cộng** | **57** |
