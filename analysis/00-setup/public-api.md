# LOTUS Public API

> **lotus-ai v1.1.4** -- Danh sách đầy đủ tất cả phương thức public API.

## Semantic Operators (DataFrame Accessors)

> Gọi qua `df.sem_xxx(...)`. Đây là API chính của LOTUS.

| Method | Signature | File:Line | Return Type | Mô tả |
|--------|-----------|-----------|-------------|-------|
| `sem_filter` | `(user_instruction, return_raw_outputs=False, return_explanations=False, return_all=False, default=True, suffix="_filter", examples=None, helper_examples=None, strategy=None, cascade_args=None, return_stats=False, safe_mode=False, progress_bar_desc="Filtering", additional_cot_instructions="")` | `lotus/sem_ops/sem_filter.py:334` | `pd.DataFrame` | Lọc hàng trong DataFrame theo điều kiện ngôn ngữ tự nhiên. Hỗ trợ cascade optimization với helper_lm. |
| `sem_map` | `(user_instruction, system_prompt=None, postprocessor=map_postprocess, return_explanations=False, return_raw_outputs=False, suffix="_map", examples=None, strategy=None, safe_mode=False, progress_bar_desc="Mapping", **model_kwargs)` | `lotus/sem_ops/sem_map.py:215` | `pd.DataFrame` | Tạo cột mới bằng cách áp dụng LLM instruction lên từng hàng. Hỗ trợ few-shot examples. |
| `sem_join` | `(other, join_instruction, return_explanations=False, how="inner", suffix="_join", examples=None, strategy=None, default=True, cascade_args=None, return_stats=False, safe_mode=False, progress_bar_desc="Join comparisons")` | `lotus/sem_ops/sem_join.py:670` | `pd.DataFrame` | Join 2 DataFrame theo điều kiện ngữ nghĩa. Hỗ trợ inner/left/right/outer join. |
| `sem_agg` | `(user_instruction, all_cols=False, suffix="_output", group_by=None, safe_mode=False, progress_bar_desc="Aggregating", long_context_strategy=LongContextStrategy.CHUNK)` | `lotus/sem_ops/sem_agg.py:354` | `pd.DataFrame` | Tổng hợp nhiều hàng thành một kết quả bằng LLM. Hỗ trợ group_by và xử lý văn bản dài. |
| `sem_topk` | `(user_instruction, K, method="quick", strategy=None, group_by=None, cascade_threshold=None, return_stats=False, safe_mode=False, return_explanations=False)` | `lotus/sem_ops/sem_topk.py:735` | `pd.DataFrame` | Chọn K hàng tốt nhất theo tiêu chí ngữ nghĩa. Dùng quicksort-style algorithm. |
| `sem_extract` | `(input_cols, output_cols, extract_quotes=False, postprocessor=extract_postprocess, return_raw_outputs=False, safe_mode=False, progress_bar_desc="Extracting", return_explanations=False, strategy=None)` | `lotus/sem_ops/sem_extract.py:202` | `pd.DataFrame` | Trích xuất thông tin có cấu trúc từ cột văn bản sang các cột mới. |
| `sem_search` | `(col_name, query, K=None, n_rerank=None, return_scores=False, suffix="_sim_score")` | `lotus/sem_ops/sem_search.py:92` | `pd.DataFrame` | Tìm kiếm tương đồng vector trên cột đã index. Hỗ trợ reranking. |
| `sem_sim_join` | `(other, left_on, right_on, K, lsuffix="", rsuffix="", score_suffix="", keep_index=False)` | `lotus/sem_ops/sem_sim_join.py:85` | `pd.DataFrame` | Join 2 DataFrame theo độ tương đồng vector, trả về K kết quả gần nhất cho mỗi hàng. |
| `sem_dedup` | `(col_name, threshold)` | `lotus/sem_ops/sem_dedup.py:33` | `pd.DataFrame` | Loại bỏ các hàng trùng lặp dựa trên độ tương đồng vector vượt threshold. |
| `sem_index` | `(col_name, index_dir)` | `lotus/sem_ops/sem_index.py:62` | `pd.DataFrame` | Tạo vector index (embeddings) cho một cột và lưu vào thư mục. |
| `load_sem_index` | `(col_name, index_dir)` | `lotus/sem_ops/load_sem_index.py:49` | `pd.DataFrame` | Tải lại vector index đã lưu trước đó từ thư mục. |
| `sem_partition_by` | `(partition_fn)` | `lotus/sem_ops/sem_partition_by.py:61` | `pd.DataFrame` | Phân vùng DataFrame theo hàm partition tự định nghĩa. |
| `sem_cluster_by` | `(col_name, ncentroids, return_scores=False, return_centroids=False, niter=20, verbose=False)` | `lotus/sem_ops/sem_cluster_by.py:58` | `pd.DataFrame` | Phân cụm DataFrame theo vector embeddings dùng FAISS k-means. |

## Evaluation Operators (DataFrame Accessors)

| Method | Signature | File:Line | Return Type | Mô tả |
|--------|-----------|-----------|-------------|-------|
| `llm_as_judge` | `(judge_instruction, response_format=None, n_trials=1, ...)` | `lotus/evals/llm_as_judge.py:188` | `pd.DataFrame` | Dùng LLM làm giám khảo đánh giá nội dung trong DataFrame. |
| `pairwise_judge` | `(col1, col2, judge_instruction, ...)` | `lotus/evals/pairwise_judge.py:70` | `pd.DataFrame` | So sánh cặp đôi 2 cột bằng LLM để xác định cột nào tốt hơn. |

## Configuration API

| Method / Attribute | Signature | File:Line | Type | Mô tả |
|--------------------|-----------|-----------|------|-------|
| `lotus.settings` | -- | `lotus/settings.py:8` | `Settings` | Singleton cấu hình toàn cục |
| `Settings.lm` | attribute | `lotus/settings.py` | `LM \| None` | Language Model chính |
| `Settings.rm` | attribute | `lotus/settings.py` | `RM \| None` | Retrieval Model |
| `Settings.helper_lm` | attribute | `lotus/settings.py` | `LM \| None` | LM phụ cho cascade optimization |
| `Settings.reranker` | attribute | `lotus/settings.py` | `Reranker \| None` | Reranker cho search |
| `Settings.vs` | attribute | `lotus/settings.py` | `VS \| None` | Vector Store backend |
| `Settings.enable_cache` | attribute | `lotus/settings.py` | `bool` | Bật/tắt cache |
| `Settings.serialization_format` | attribute | `lotus/settings.py` | `SerializationFormat` | Định dạng serialize dữ liệu |

## Model API

| Class | Constructor | File:Line | Mô tả |
|-------|-------------|-----------|-------|
| `LM` | `LM(model, ...)` | `lotus/models/lm.py:41` | Tạo LM wrapper; model là tên litellm model (e.g. "gpt-4o") |
| `SentenceTransformersRM` | `SentenceTransformersRM(model, ...)` | `lotus/models/sentence_transformers_rm.py:11` | Tạo RM từ sentence-transformers model |
| `LiteLLMRM` | `LiteLLMRM(model, ...)` | `lotus/models/litellm_rm.py:11` | Tạo RM từ litellm embeddings API |
| `ColBERTv2RM` | `ColBERTv2RM(url, ...)` | `lotus/models/colbertv2_rm.py:17` | Tạo RM kết nối ColBERTv2 server |
| `CrossEncoderReranker` | `CrossEncoderReranker(model, ...)` | `lotus/models/cross_encoder_reranker.py:7` | Tạo Reranker từ cross-encoder model |

## Vector Store API

| Class | Constructor | File:Line | Mô tả |
|-------|-------------|-----------|-------|
| `FaissVS` | `FaissVS(...)` | `lotus/vector_store/faiss_vs.py:13` | Vector store FAISS (mặc định, local) |
| `WeaviateVS` | `WeaviateVS(...)` | `lotus/vector_store/weaviate_vs.py` | Vector store Weaviate |
| `QdrantVS` | `QdrantVS(...)` | `lotus/vector_store/qdrant_vs.py` | Vector store Qdrant |

## Utility Functions

| Function | Signature | File | Return Type | Mô tả |
|----------|-----------|------|-------------|-------|
| `web_search` | `web_search(query, engine, ...)` | `lotus/web_search.py` | `WebSearchCorpus` | Tìm kiếm web từ nhiều nguồn (Google, Arxiv, You, Tavily, PubMed) |
| `web_extract` | `web_extract(url, ...)` | `lotus/web_search.py` | `str` | Trích xuất nội dung từ URL |
| `parse_cols` | `parse_cols(expression)` | `lotus/nl_expression.py` | `list[str]` | Trích xuất tên cột {col} từ langex expression |
| `calculate_cost_from_response` | `calculate_cost_from_response(response)` | `lotus/pricing.py` | `float` | Tính chi phí API call từ litellm response |
| `cluster` | `cluster(...)` | `lotus/utils.py` | -- | Phân cụm dữ liệu |
| `fetch_image` | `fetch_image(url)` | `lotus/utils.py` | -- | Tải hình ảnh từ URL |
| `show_safe_mode` | `show_safe_mode()` | `lotus/utils.py` | -- | Hiển thị thông báo safe mode |

## Template Functions

| Function | Signature | File | Mô tả |
|----------|-----------|------|-------|
| `filter_formatter` | `filter_formatter(...)` | `lotus/templates/task_instructions.py` | Tạo prompt cho sem_filter |
| `map_formatter` | `map_formatter(...)` | `lotus/templates/task_instructions.py` | Tạo prompt cho sem_map |
| `extract_formatter` | `extract_formatter(...)` | `lotus/templates/task_instructions.py` | Tạo prompt cho sem_extract |
| `df2text` | `df2text(df, cols)` | `lotus/templates/task_instructions.py` | Chuyển DataFrame thành văn bản cho prompt |
| `df2multimodal_info` | `df2multimodal_info(df, cols)` | `lotus/templates/task_instructions.py` | Chuyển DataFrame thành thông tin multimodal (text + image) |

## __all__ Export List

```python
__all__ = [
    "sem_map", "sem_filter", "sem_agg", "sem_extract", "sem_join",
    "sem_partition_by", "sem_topk", "sem_index", "load_sem_index",
    "sem_sim_join", "sem_cluster_by", "sem_search", "sem_dedup",
    "settings", "nl_expression", "templates", "logger",
    "models", "vector_store", "utils", "dtype_extensions",
    "web_search", "web_extract", "WebSearchCorpus",
    "llm_as_judge", "pairwise_judge",
]
```
