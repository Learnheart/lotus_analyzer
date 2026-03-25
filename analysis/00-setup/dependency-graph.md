# LOTUS Dependency Graph

> **lotus-ai v1.1.4** -- Phân tích hệ thống dependencies và build system.

## Build System

| Thuộc tính | Giá trị |
|------------|---------|
| Build backend | `hatchling` |
| Package name | `lotus-ai` |
| Version | `1.1.4` |
| Python requirement | (xem pyproject.toml) |

## Core Dependencies (Bắt buộc)

> Các thư viện luôn được cài đặt khi `pip install lotus-ai`.

| Package | Vai trò trong LOTUS |
|---------|---------------------|
| `backoff` | Retry logic cho API calls (rate limiting, transient errors) |
| `faiss-cpu` | Vector similarity search -- backend mặc định cho FaissVS |
| `litellm` | Unified LLM API -- LM wrapper gọi nhiều provider (OpenAI, Anthropic, v.v.) |
| `numpy` | Xử lý mảng số -- embeddings, vector operations |
| `pandas` | Core data structure -- DataFrame là trung tâm của LOTUS |
| `sentence-transformers` | Embedding models -- SentenceTransformersRM |
| `tiktoken` | Token counting -- đếm token cho OpenAI models |
| `tqdm` | Progress bars -- hiển thị tiến trình các sem_ops |
| `pydantic` | Data validation -- định nghĩa các type classes |

## Optional Dependencies

### `xml` -- Hỗ trợ serialization XML
| Package | Vai trò |
|---------|---------|
| `lxml` | Parse và tạo XML output từ LLM |

### `web_search` -- Tìm kiếm web
| Package | Vai trò |
|---------|---------|
| `serpapi` | Google Search API |
| `arxiv` | ArXiv paper search |
| `pymed` | PubMed article search |
| `requests` | HTTP requests chung |
| `azure` | Azure search integration |
| `tavily-python` | Tavily search API |

### `file_extractor` -- Đọc nhiều định dạng file
| Package | Vai trò |
|---------|---------|
| `llama-index` | Framework đọc file |
| `pymupdf` | Đọc file PDF |
| `docx2txt` | Đọc file Word (.docx) |
| `python-pptx` | Đọc file PowerPoint (.pptx) |
| `python-magic` | Nhận diện loại file từ magic bytes |

### `data_connectors` -- Kết nối nguồn dữ liệu
| Package | Vai trò |
|---------|---------|
| `sqlalchemy` | Kết nối SQL databases |
| `boto3` | Kết nối AWS S3 |

### Vector Store Backends
| Package | Vai trò |
|---------|---------|
| `weaviate-client` | Client cho WeaviateVS |
| `qdrant-client` | Client cho QdrantVS |

## Dev Dependencies

| Package | Vai trò |
|---------|---------|
| `ruff` | Linter + formatter (thay thế flake8, black, isort) |
| `mypy` | Static type checking |
| `pytest` | Testing framework |
| `pre-commit` | Git hooks tự động kiểm tra code trước commit |

## Internal Dependency Graph

> Biểu đồ phụ thuộc giữa các module nội bộ của LOTUS.

```
lotus/__init__.py
  |-- lotus/settings.py          (import Settings)
  |-- lotus/sem_ops/*            (import tất cả sem operators để đăng ký accessors)
  |-- lotus/evals/*              (import llm_as_judge, pairwise_judge)
  |-- lotus/models/*             (re-export)
  |-- lotus/vector_store/*       (re-export)
  |-- lotus/web_search.py        (re-export)

lotus/sem_ops/sem_filter.py
  |-- lotus/settings.py          (lấy lm, helper_lm từ settings)
  |-- lotus/models/lm.py         (gọi LM.generate)
  |-- lotus/templates/           (tạo prompt)
  |-- lotus/types.py             (SemanticFilterOutput, CascadeArgs, v.v.)
  |-- lotus/nl_expression.py     (parse_cols)
  |-- lotus/sem_ops/cascade_utils.py  (cascade optimization)
  |-- lotus/sem_ops/postprocessors.py (xử lý output)
  |-- lotus/cache.py             (operator_cache decorator)

lotus/sem_ops/sem_map.py
  |-- lotus/settings.py
  |-- lotus/models/lm.py
  |-- lotus/templates/
  |-- lotus/types.py
  |-- lotus/nl_expression.py
  |-- lotus/sem_ops/postprocessors.py
  |-- lotus/cache.py

lotus/sem_ops/sem_join.py
  |-- lotus/settings.py
  |-- lotus/models/lm.py
  |-- lotus/templates/
  |-- lotus/types.py
  |-- lotus/nl_expression.py
  |-- lotus/sem_ops/cascade_utils.py
  |-- lotus/sem_ops/postprocessors.py
  |-- lotus/cache.py

lotus/sem_ops/sem_agg.py
  |-- lotus/settings.py
  |-- lotus/models/lm.py
  |-- lotus/templates/
  |-- lotus/types.py
  |-- lotus/long_context_strategy.py  (xử lý văn bản dài)
  |-- lotus/cache.py

lotus/sem_ops/sem_topk.py
  |-- lotus/settings.py
  |-- lotus/models/lm.py
  |-- lotus/templates/
  |-- lotus/types.py
  |-- lotus/sem_ops/cascade_utils.py

lotus/sem_ops/sem_search.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py          (retrieval)
  |-- lotus/models/reranker.py    (reranking)
  |-- lotus/vector_store/         (truy vấn index)

lotus/sem_ops/sem_index.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py
  |-- lotus/vector_store/         (tạo index)

lotus/sem_ops/sem_sim_join.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py
  |-- lotus/vector_store/

lotus/sem_ops/sem_dedup.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py

lotus/sem_ops/sem_cluster_by.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py
  |-- faiss                       (k-means clustering)

lotus/models/lm.py
  |-- litellm                     (API calls)
  |-- tiktoken                    (token counting)
  |-- lotus/types.py
  |-- lotus/cache.py
  |-- lotus/pricing.py

lotus/vector_store/faiss_vs.py
  |-- faiss                       (FAISS index)
  |-- numpy

lotus/dtype_extensions/image.py
  |-- pandas                      (ExtensionDtype, ExtensionArray)
  |-- lotus/utils.py              (fetch_image)
```

## Dependency Layers

```
Layer 4: User Code
  |  df.sem_filter("..."), df.sem_map("..."), v.v.
  |
Layer 3: Semantic Operators (lotus/sem_ops/)
  |  Đăng ký như pandas DataFrame accessors
  |  Phụ thuộc: templates, models, types, cache, nl_expression
  |
Layer 2: Models & Infrastructure
  |  LM (litellm), RM (sentence-transformers), VS (faiss)
  |  Cache (memory/sqlite), Templates (prompt formatting)
  |
Layer 1: Core Types & Settings
  |  types.py, settings.py, nl_expression.py, utils.py
  |
Layer 0: External Libraries
     pandas, litellm, faiss-cpu, sentence-transformers, numpy, pydantic
```

## Ghi chú quan trọng

1. **litellm là trung tâm**: Tất cả LLM calls đi qua litellm, cho phép LOTUS hỗ trợ 100+ LLM providers mà không cần thay đổi code.

2. **pandas là nền tảng**: Mỗi sem_op là một DataFrame accessor, nghĩa là LOTUS mở rộng pandas thay vì thay thế nó.

3. **faiss-cpu là bắt buộc**: Mặc dù có WeaviateVS và QdrantVS, faiss-cpu là core dependency vì FaissVS là vector store mặc định và sem_cluster_by dùng FAISS k-means trực tiếp.

4. **sentence-transformers là bắt buộc**: Dù có LiteLLMRM và ColBERTv2RM, sentence-transformers vẫn là core dependency vì SentenceTransformersRM là RM mặc định.

5. **Cascade pattern**: sem_filter, sem_join, sem_topk hỗ trợ cascade optimization -- dùng helper_lm (rẻ/nhanh) trước, chỉ gọi lm chính khi cần, giảm chi phí.
