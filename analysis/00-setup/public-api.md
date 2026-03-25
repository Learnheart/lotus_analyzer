# LOTUS Public API

> **lotus-ai v1.1.4** -- Danh sach day du tat ca phuong thuc public API.

## Semantic Operators (DataFrame Accessors)

> Goi qua `df.sem_xxx(...)`. Day la API chinh cua LOTUS.

| Method | Signature | File:Line | Return Type | Mo ta |
|--------|-----------|-----------|-------------|-------|
| `sem_filter` | `(user_instruction, return_raw_outputs=False, return_explanations=False, return_all=False, default=True, suffix="_filter", examples=None, helper_examples=None, strategy=None, cascade_args=None, return_stats=False, safe_mode=False, progress_bar_desc="Filtering", additional_cot_instructions="")` | `lotus/sem_ops/sem_filter.py:334` | `pd.DataFrame` | Loc hang trong DataFrame theo dieu kien ngon ngu tu nhien. Ho tro cascade optimization voi helper_lm. |
| `sem_map` | `(user_instruction, system_prompt=None, postprocessor=map_postprocess, return_explanations=False, return_raw_outputs=False, suffix="_map", examples=None, strategy=None, safe_mode=False, progress_bar_desc="Mapping", **model_kwargs)` | `lotus/sem_ops/sem_map.py:215` | `pd.DataFrame` | Tao cot moi bang cach ap dung LLM instruction len tung hang. Ho tro few-shot examples. |
| `sem_join` | `(other, join_instruction, return_explanations=False, how="inner", suffix="_join", examples=None, strategy=None, default=True, cascade_args=None, return_stats=False, safe_mode=False, progress_bar_desc="Join comparisons")` | `lotus/sem_ops/sem_join.py:670` | `pd.DataFrame` | Join 2 DataFrame theo dieu kien ngu nghia. Ho tro inner/left/right/outer join. |
| `sem_agg` | `(user_instruction, all_cols=False, suffix="_output", group_by=None, safe_mode=False, progress_bar_desc="Aggregating", long_context_strategy=LongContextStrategy.CHUNK)` | `lotus/sem_ops/sem_agg.py:354` | `pd.DataFrame` | Tong hop nhieu hang thanh mot ket qua bang LLM. Ho tro group_by va xu ly van ban dai. |
| `sem_topk` | `(user_instruction, K, method="quick", strategy=None, group_by=None, cascade_threshold=None, return_stats=False, safe_mode=False, return_explanations=False)` | `lotus/sem_ops/sem_topk.py:735` | `pd.DataFrame` | Chon K hang tot nhat theo tieu chi ngu nghia. Dung quicksort-style algorithm. |
| `sem_extract` | `(input_cols, output_cols, extract_quotes=False, postprocessor=extract_postprocess, return_raw_outputs=False, safe_mode=False, progress_bar_desc="Extracting", return_explanations=False, strategy=None)` | `lotus/sem_ops/sem_extract.py:202` | `pd.DataFrame` | Trich xuat thong tin co cau truc tu cot van ban sang cac cot moi. |
| `sem_search` | `(col_name, query, K=None, n_rerank=None, return_scores=False, suffix="_sim_score")` | `lotus/sem_ops/sem_search.py:92` | `pd.DataFrame` | Tim kiem tuong dong vector tren cot da index. Ho tro reranking. |
| `sem_sim_join` | `(other, left_on, right_on, K, lsuffix="", rsuffix="", score_suffix="", keep_index=False)` | `lotus/sem_ops/sem_sim_join.py:85` | `pd.DataFrame` | Join 2 DataFrame theo do tuong dong vector, tra ve K ket qua gan nhat cho moi hang. |
| `sem_dedup` | `(col_name, threshold)` | `lotus/sem_ops/sem_dedup.py:33` | `pd.DataFrame` | Loai bo cac hang trung lap dua tren do tuong dong vector vuot threshold. |
| `sem_index` | `(col_name, index_dir)` | `lotus/sem_ops/sem_index.py:62` | `pd.DataFrame` | Tao vector index (embeddings) cho mot cot va luu vao thu muc. |
| `load_sem_index` | `(col_name, index_dir)` | `lotus/sem_ops/load_sem_index.py:49` | `pd.DataFrame` | Tai lai vector index da luu truoc do tu thu muc. |
| `sem_partition_by` | `(partition_fn)` | `lotus/sem_ops/sem_partition_by.py:61` | `pd.DataFrame` | Phan vung DataFrame theo ham partition tu dinh nghia. |
| `sem_cluster_by` | `(col_name, ncentroids, return_scores=False, return_centroids=False, niter=20, verbose=False)` | `lotus/sem_ops/sem_cluster_by.py:58` | `pd.DataFrame` | Phan cum DataFrame theo vector embeddings dung FAISS k-means. |

## Evaluation Operators (DataFrame Accessors)

| Method | Signature | File:Line | Return Type | Mo ta |
|--------|-----------|-----------|-------------|-------|
| `llm_as_judge` | `(judge_instruction, response_format=None, n_trials=1, ...)` | `lotus/evals/llm_as_judge.py:188` | `pd.DataFrame` | Dung LLM lam giam khao danh gia noi dung trong DataFrame. |
| `pairwise_judge` | `(col1, col2, judge_instruction, ...)` | `lotus/evals/pairwise_judge.py:70` | `pd.DataFrame` | So sanh cap doi 2 cot bang LLM de xac dinh cot nao tot hon. |

## Configuration API

| Method / Attribute | Signature | File:Line | Type | Mo ta |
|--------------------|-----------|-----------|------|-------|
| `lotus.settings` | -- | `lotus/settings.py:8` | `Settings` | Singleton cau hinh toan cuc |
| `Settings.lm` | attribute | `lotus/settings.py` | `LM \| None` | Language Model chinh |
| `Settings.rm` | attribute | `lotus/settings.py` | `RM \| None` | Retrieval Model |
| `Settings.helper_lm` | attribute | `lotus/settings.py` | `LM \| None` | LM phu cho cascade optimization |
| `Settings.reranker` | attribute | `lotus/settings.py` | `Reranker \| None` | Reranker cho search |
| `Settings.vs` | attribute | `lotus/settings.py` | `VS \| None` | Vector Store backend |
| `Settings.enable_cache` | attribute | `lotus/settings.py` | `bool` | Bat/tat cache |
| `Settings.serialization_format` | attribute | `lotus/settings.py` | `SerializationFormat` | Dinh dang serialize du lieu |

## Model API

| Class | Constructor | File:Line | Mo ta |
|-------|-------------|-----------|-------|
| `LM` | `LM(model, ...)` | `lotus/models/lm.py:41` | Tao LM wrapper; model la ten litellm model (e.g. "gpt-4o") |
| `SentenceTransformersRM` | `SentenceTransformersRM(model, ...)` | `lotus/models/sentence_transformers_rm.py:11` | Tao RM tu sentence-transformers model |
| `LiteLLMRM` | `LiteLLMRM(model, ...)` | `lotus/models/litellm_rm.py:11` | Tao RM tu litellm embeddings API |
| `ColBERTv2RM` | `ColBERTv2RM(url, ...)` | `lotus/models/colbertv2_rm.py:17` | Tao RM ket noi ColBERTv2 server |
| `CrossEncoderReranker` | `CrossEncoderReranker(model, ...)` | `lotus/models/cross_encoder_reranker.py:7` | Tao Reranker tu cross-encoder model |

## Vector Store API

| Class | Constructor | File:Line | Mo ta |
|-------|-------------|-----------|-------|
| `FaissVS` | `FaissVS(...)` | `lotus/vector_store/faiss_vs.py:13` | Vector store FAISS (mac dinh, local) |
| `WeaviateVS` | `WeaviateVS(...)` | `lotus/vector_store/weaviate_vs.py` | Vector store Weaviate |
| `QdrantVS` | `QdrantVS(...)` | `lotus/vector_store/qdrant_vs.py` | Vector store Qdrant |

## Utility Functions

| Function | Signature | File | Return Type | Mo ta |
|----------|-----------|------|-------------|-------|
| `web_search` | `web_search(query, engine, ...)` | `lotus/web_search.py` | `WebSearchCorpus` | Tim kiem web tu nhieu nguon (Google, Arxiv, You, Tavily, PubMed) |
| `web_extract` | `web_extract(url, ...)` | `lotus/web_search.py` | `str` | Trich xuat noi dung tu URL |
| `parse_cols` | `parse_cols(expression)` | `lotus/nl_expression.py` | `list[str]` | Trich xuat ten cot {col} tu langex expression |
| `calculate_cost_from_response` | `calculate_cost_from_response(response)` | `lotus/pricing.py` | `float` | Tinh chi phi API call tu litellm response |
| `cluster` | `cluster(...)` | `lotus/utils.py` | -- | Phan cum du lieu |
| `fetch_image` | `fetch_image(url)` | `lotus/utils.py` | -- | Tai hinh anh tu URL |
| `show_safe_mode` | `show_safe_mode()` | `lotus/utils.py` | -- | Hien thi thong bao safe mode |

## Template Functions

| Function | Signature | File | Mo ta |
|----------|-----------|------|-------|
| `filter_formatter` | `filter_formatter(...)` | `lotus/templates/task_instructions.py` | Tao prompt cho sem_filter |
| `map_formatter` | `map_formatter(...)` | `lotus/templates/task_instructions.py` | Tao prompt cho sem_map |
| `extract_formatter` | `extract_formatter(...)` | `lotus/templates/task_instructions.py` | Tao prompt cho sem_extract |
| `df2text` | `df2text(df, cols)` | `lotus/templates/task_instructions.py` | Chuyen DataFrame thanh van ban cho prompt |
| `df2multimodal_info` | `df2multimodal_info(df, cols)` | `lotus/templates/task_instructions.py` | Chuyen DataFrame thanh thong tin multimodal (text + image) |

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
