# LOTUS Repository Map

> **lotus-ai v1.1.4** -- Cây thư mục và mô tả chức năng của từng thành phần.
> Build system: hatchling | Python package

## Directory Tree

```
lotus/                              # Root package -- thư viện chính
|-- __init__.py                     # Entry point: export tất cả public API (__all__), đăng ký sem_ops accessors
|-- settings.py                     # Lớp Settings: cấu hình toàn cục (lm, rm, helper_lm, reranker, vs, enable_cache, serialization_format)
|-- types.py                        # Định nghĩa tất cả data types: LMOutput, LMStats, CascadeArgs, SemanticFilterOutput, v.v.
|-- cache.py                        # Hệ thống cache: InMemoryCache, SQLiteCache, CacheFactory, operator_cache decorator
|-- nl_expression.py                # parse_cols() -- trích xuất {col} từ langex (natural language expression)
|-- pricing.py                      # calculate_cost_from_response() -- tính chi phí sử dụng litellm
|-- utils.py                        # Tiện ích chung: cluster(), fetch_image(), show_safe_mode()
|-- long_context_strategy.py        # Xử lý văn bản dài: ChunkedDocument, create_chunked_documents, TRUNCATE/CHUNK strategies
|-- web_search.py                   # Tìm kiếm web: web_search (Google, Arxiv, You, Tavily, PubMed), web_extract
|
|-- models/                         # Các lớp mô hình AI -- abstraction layer cho LLM và retrieval
|   |-- __init__.py                 # Export các lớp model
|   |-- lm.py                      # LM: Language Model wrapper (litellm-based), xử lý batching, caching, token counting
|   |-- rm.py                      # RM: Retrieval Model (abstract base class)
|   |-- reranker.py                # Reranker: abstract base class cho re-ranking
|   |-- sentence_transformers_rm.py # SentenceTransformersRM: RM dùng sentence-transformers embeddings
|   |-- litellm_rm.py              # LiteLLMRM: RM dùng litellm embeddings API
|   |-- colbertv2_rm.py            # ColBERTv2RM: RM dùng ColBERTv2 server
|   |-- cross_encoder_reranker.py  # CrossEncoderReranker: Reranker dùng cross-encoder model
|
|-- sem_ops/                        # Semantic operators -- cốt lõi của LOTUS, đăng ký như pandas DataFrame accessors
|   |-- __init__.py                 # Export tất cả sem_ops
|   |-- sem_filter.py              # sem_filter: lọc DataFrame theo điều kiện ngôn ngữ tự nhiên
|   |-- sem_map.py                 # sem_map: ánh xạ/chuyển đổi cột bằng LLM
|   |-- sem_join.py                # sem_join: join 2 DataFrame theo ngữ nghĩa (semantic join)
|   |-- sem_agg.py                 # sem_agg: tổng hợp dữ liệu bằng LLM (aggregate)
|   |-- sem_topk.py                # sem_topk: chọn top-K hàng theo tiêu chí ngôn ngữ tự nhiên
|   |-- sem_extract.py             # sem_extract: trích xuất thông tin từ cột sang cột mới
|   |-- sem_search.py              # sem_search: tìm kiếm vector + reranking
|   |-- sem_sim_join.py            # sem_sim_join: join theo độ tương đồng vector (similarity join)
|   |-- sem_dedup.py               # sem_dedup: loại bỏ bản sao theo độ tương đồng (deduplication)
|   |-- sem_index.py               # sem_index: tạo index vector cho cột
|   |-- sem_partition_by.py        # sem_partition_by: phân vùng DataFrame theo hàm partition
|   |-- sem_cluster_by.py          # sem_cluster_by: phân cụm DataFrame theo vector embeddings
|   |-- cascade_utils.py           # Tiện ích cho cascade optimization (helper model + main model)
|   |-- postprocessors.py          # Hậu xử lý output từ LLM (map_postprocess, extract_postprocess, v.v.)
|   |-- load_sem_index.py          # load_sem_index: tải lại index đã lưu trước đó
|
|-- templates/                      # Prompt templates -- định dạng prompt gửi đến LLM
|   |-- __init__.py                 # Export templates
|   |-- task_instructions.py       # filter_formatter, map_formatter, extract_formatter, df2text, df2multimodal_info
|
|-- vector_store/                   # Vector store backends -- lưu trữ và truy vấn vector embeddings
|   |-- __init__.py                 # Export vector stores
|   |-- vs.py                      # VS: abstract base class cho vector store
|   |-- faiss_vs.py                # FaissVS: vector store dùng FAISS (mặc định)
|   |-- weaviate_vs.py             # WeaviateVS: vector store dùng Weaviate
|   |-- qdrant_vs.py               # QdrantVS: vector store dùng Qdrant
|
|-- dtype_extensions/               # Pandas custom dtype -- mở rộng kiểu dữ liệu pandas
|   |-- __init__.py                 # Export dtype extensions
|   |-- image.py                   # ImageDtype + ImageArray: kiểu dữ liệu hình ảnh cho pandas DataFrame
|
|-- evals/                          # Evaluation tools -- đánh giá chất lượng bằng LLM
|   |-- __init__.py                 # Export eval functions
|   |-- llm_as_judge.py           # llm_as_judge: dùng LLM làm giám khảo đánh giá
|   |-- pairwise_judge.py         # pairwise_judge: so sánh cặp đôi (pairwise comparison)
|
|-- data_connectors/                # Kết nối dữ liệu -- đọc dữ liệu từ nhiều nguồn
|   |-- __init__.py                 # Export connectors
|   |-- connectors.py             # Connectors: sqlalchemy, boto3, v.v.
|
|-- file_extractors/                # Trích xuất file -- đọc nội dung từ nhiều định dạng
|   |-- __init__.py                 # Export extractors
|   |-- directory_reader.py        # Đọc thư mục và các file bên trong
|   |-- pptx.py                    # Trích xuất nội dung từ file PowerPoint (.pptx)
```

## Kiến trúc tổng quan

```
                    +-------------------+
                    |   lotus.settings  |  <-- Cấu hình toàn cục (LM, RM, VS, cache)
                    +-------------------+
                             |
                    +-------------------+
                    |   lotus.models    |  <-- LM, RM, Reranker (abstraction)
                    +-------------------+
                        |           |
              +---------+           +-----------+
              |                                 |
    +-------------------+             +-------------------+
    |   lotus.sem_ops   |             | lotus.vector_store|
    | (DataFrame        |             | (FAISS, Weaviate, |
    |  accessors)       |             |  Qdrant)          |
    +-------------------+             +-------------------+
              |
    +-------------------+
    | lotus.templates   |  <-- Prompt formatting
    +-------------------+
```

**Nguyên tắc hoạt động**: Người dùng gọi `df.sem_filter(...)`, `df.sem_map(...)`, v.v. trên pandas DataFrame.
Các operator này sử dụng `lotus.settings` để lấy model (LM/RM), tạo prompt từ `lotus.templates`,
gọi LLM qua `lotus.models.LM`, và trả kết quả về dạng DataFrame mới.
