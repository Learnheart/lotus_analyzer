# LOTUS Repository Map

> **lotus-ai v1.1.4** -- Cay thu muc va mo ta chuc nang cua tung thanh phan.
> Build system: hatchling | Python package

## Directory Tree

```
lotus/                              # Root package -- thu vien chinh
|-- __init__.py                     # Entry point: export tat ca public API (__all__), dang ky sem_ops accessors
|-- settings.py                     # Lop Settings: cau hinh toan cuc (lm, rm, helper_lm, reranker, vs, enable_cache, serialization_format)
|-- types.py                        # Dinh nghia tat ca data types: LMOutput, LMStats, CascadeArgs, SemanticFilterOutput, v.v.
|-- cache.py                        # He thong cache: InMemoryCache, SQLiteCache, CacheFactory, operator_cache decorator
|-- nl_expression.py                # parse_cols() -- trich xuat {col} tu langex (natural language expression)
|-- pricing.py                      # calculate_cost_from_response() -- tinh chi phi su dung litellm
|-- utils.py                        # Tien ich chung: cluster(), fetch_image(), show_safe_mode()
|-- long_context_strategy.py        # Xu ly van ban dai: ChunkedDocument, create_chunked_documents, TRUNCATE/CHUNK strategies
|-- web_search.py                   # Tim kiem web: web_search (Google, Arxiv, You, Tavily, PubMed), web_extract
|
|-- models/                         # Cac lop mo hinh AI -- abstraction layer cho LLM va retrieval
|   |-- __init__.py                 # Export cac lop model
|   |-- lm.py                      # LM: Language Model wrapper (litellm-based), xu ly batching, caching, token counting
|   |-- rm.py                      # RM: Retrieval Model (abstract base class)
|   |-- reranker.py                # Reranker: abstract base class cho re-ranking
|   |-- sentence_transformers_rm.py # SentenceTransformersRM: RM dung sentence-transformers embeddings
|   |-- litellm_rm.py              # LiteLLMRM: RM dung litellm embeddings API
|   |-- colbertv2_rm.py            # ColBERTv2RM: RM dung ColBERTv2 server
|   |-- cross_encoder_reranker.py  # CrossEncoderReranker: Reranker dung cross-encoder model
|
|-- sem_ops/                        # Semantic operators -- cot loi cua LOTUS, dang ky nhu pandas DataFrame accessors
|   |-- __init__.py                 # Export tat ca sem_ops
|   |-- sem_filter.py              # sem_filter: loc DataFrame theo dieu kien ngon ngu tu nhien
|   |-- sem_map.py                 # sem_map: anh xa/chuyen doi cot bang LLM
|   |-- sem_join.py                # sem_join: join 2 DataFrame theo ngu nghia (semantic join)
|   |-- sem_agg.py                 # sem_agg: tong hop du lieu bang LLM (aggregate)
|   |-- sem_topk.py                # sem_topk: chon top-K hang theo tieu chi ngon ngu tu nhien
|   |-- sem_extract.py             # sem_extract: trich xuat thong tin tu cot sang cot moi
|   |-- sem_search.py              # sem_search: tim kiem vector + reranking
|   |-- sem_sim_join.py            # sem_sim_join: join theo do tuong dong vector (similarity join)
|   |-- sem_dedup.py               # sem_dedup: loai bo ban sao theo do tuong dong (deduplication)
|   |-- sem_index.py               # sem_index: tao index vector cho cot
|   |-- sem_partition_by.py        # sem_partition_by: phan vung DataFrame theo ham partition
|   |-- sem_cluster_by.py          # sem_cluster_by: phan cum DataFrame theo vector embeddings
|   |-- cascade_utils.py           # Tien ich cho cascade optimization (helper model + main model)
|   |-- postprocessors.py          # Hau xu ly output tu LLM (map_postprocess, extract_postprocess, v.v.)
|   |-- load_sem_index.py          # load_sem_index: tai lai index da luu truoc do
|
|-- templates/                      # Prompt templates -- dinh dang prompt gui den LLM
|   |-- __init__.py                 # Export templates
|   |-- task_instructions.py       # filter_formatter, map_formatter, extract_formatter, df2text, df2multimodal_info
|
|-- vector_store/                   # Vector store backends -- luu tru va truy van vector embeddings
|   |-- __init__.py                 # Export vector stores
|   |-- vs.py                      # VS: abstract base class cho vector store
|   |-- faiss_vs.py                # FaissVS: vector store dung FAISS (mac dinh)
|   |-- weaviate_vs.py             # WeaviateVS: vector store dung Weaviate
|   |-- qdrant_vs.py               # QdrantVS: vector store dung Qdrant
|
|-- dtype_extensions/               # Pandas custom dtype -- mo rong kieu du lieu pandas
|   |-- __init__.py                 # Export dtype extensions
|   |-- image.py                   # ImageDtype + ImageArray: kieu du lieu hinh anh cho pandas DataFrame
|
|-- evals/                          # Evaluation tools -- danh gia chat luong bang LLM
|   |-- __init__.py                 # Export eval functions
|   |-- llm_as_judge.py           # llm_as_judge: dung LLM lam giam khao danh gia
|   |-- pairwise_judge.py         # pairwise_judge: so sanh cap doi (pairwise comparison)
|
|-- data_connectors/                # Ket noi du lieu -- doc du lieu tu nhieu nguon
|   |-- __init__.py                 # Export connectors
|   |-- connectors.py             # Connectors: sqlalchemy, boto3, v.v.
|
|-- file_extractors/                # Trich xuat file -- doc noi dung tu nhieu dinh dang
|   |-- __init__.py                 # Export extractors
|   |-- directory_reader.py        # Doc thu muc va cac file ben trong
|   |-- pptx.py                    # Trich xuat noi dung tu file PowerPoint (.pptx)
```

## Kien truc tong quan

```
                    +-------------------+
                    |   lotus.settings  |  <-- Cau hinh toan cuc (LM, RM, VS, cache)
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

**Nguyen tac hoat dong**: Nguoi dung goi `df.sem_filter(...)`, `df.sem_map(...)`, v.v. tren pandas DataFrame.
Cac operator nay su dung `lotus.settings` de lay model (LM/RM), tao prompt tu `lotus.templates`,
goi LLM qua `lotus.models.LM`, va tra ket qua ve dang DataFrame moi.
