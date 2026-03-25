# LOTUS Dependency Graph

> **lotus-ai v1.1.4** -- Phan tich he thong dependencies va build system.

## Build System

| Thuoc tinh | Gia tri |
|------------|---------|
| Build backend | `hatchling` |
| Package name | `lotus-ai` |
| Version | `1.1.4` |
| Python requirement | (xem pyproject.toml) |

## Core Dependencies (Bat buoc)

> Cac thu vien luon duoc cai dat khi `pip install lotus-ai`.

| Package | Vai tro trong LOTUS |
|---------|---------------------|
| `backoff` | Retry logic cho API calls (rate limiting, transient errors) |
| `faiss-cpu` | Vector similarity search -- backend mac dinh cho FaissVS |
| `litellm` | Unified LLM API -- LM wrapper goi nhieu provider (OpenAI, Anthropic, v.v.) |
| `numpy` | Xu ly mang so -- embeddings, vector operations |
| `pandas` | Core data structure -- DataFrame la trung tam cua LOTUS |
| `sentence-transformers` | Embedding models -- SentenceTransformersRM |
| `tiktoken` | Token counting -- dem token cho OpenAI models |
| `tqdm` | Progress bars -- hien thi tien trinh cac sem_ops |
| `pydantic` | Data validation -- dinh nghia cac type classes |

## Optional Dependencies

### `xml` -- Ho tro serialization XML
| Package | Vai tro |
|---------|---------|
| `lxml` | Parse va tao XML output tu LLM |

### `web_search` -- Tim kiem web
| Package | Vai tro |
|---------|---------|
| `serpapi` | Google Search API |
| `arxiv` | ArXiv paper search |
| `pymed` | PubMed article search |
| `requests` | HTTP requests chung |
| `azure` | Azure search integration |
| `tavily-python` | Tavily search API |

### `file_extractor` -- Doc nhieu dinh dang file
| Package | Vai tro |
|---------|---------|
| `llama-index` | Framework doc file |
| `pymupdf` | Doc file PDF |
| `docx2txt` | Doc file Word (.docx) |
| `python-pptx` | Doc file PowerPoint (.pptx) |
| `python-magic` | Nhan dien loai file tu magic bytes |

### `data_connectors` -- Ket noi nguon du lieu
| Package | Vai tro |
|---------|---------|
| `sqlalchemy` | Ket noi SQL databases |
| `boto3` | Ket noi AWS S3 |

### Vector Store Backends
| Package | Vai tro |
|---------|---------|
| `weaviate-client` | Client cho WeaviateVS |
| `qdrant-client` | Client cho QdrantVS |

## Dev Dependencies

| Package | Vai tro |
|---------|---------|
| `ruff` | Linter + formatter (thay the flake8, black, isort) |
| `mypy` | Static type checking |
| `pytest` | Testing framework |
| `pre-commit` | Git hooks tu dong kiem tra code truoc commit |

## Internal Dependency Graph

> Bieu do phu thuoc giua cac module noi bo cua LOTUS.

```
lotus/__init__.py
  |-- lotus/settings.py          (import Settings)
  |-- lotus/sem_ops/*            (import tat ca sem operators de dang ky accessors)
  |-- lotus/evals/*              (import llm_as_judge, pairwise_judge)
  |-- lotus/models/*             (re-export)
  |-- lotus/vector_store/*       (re-export)
  |-- lotus/web_search.py        (re-export)

lotus/sem_ops/sem_filter.py
  |-- lotus/settings.py          (lay lm, helper_lm tu settings)
  |-- lotus/models/lm.py         (goi LM.generate)
  |-- lotus/templates/           (tao prompt)
  |-- lotus/types.py             (SemanticFilterOutput, CascadeArgs, v.v.)
  |-- lotus/nl_expression.py     (parse_cols)
  |-- lotus/sem_ops/cascade_utils.py  (cascade optimization)
  |-- lotus/sem_ops/postprocessors.py (xu ly output)
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
  |-- lotus/long_context_strategy.py  (xu ly van ban dai)
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
  |-- lotus/vector_store/         (truy van index)

lotus/sem_ops/sem_index.py
  |-- lotus/settings.py
  |-- lotus/models/rm.py
  |-- lotus/vector_store/         (tao index)

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
  |  Dang ky nhu pandas DataFrame accessors
  |  Phu thuoc: templates, models, types, cache, nl_expression
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

## Ghi chu quan trong

1. **litellm la trung tam**: Tat ca LLM calls di qua litellm, cho phep LOTUS ho tro 100+ LLM providers ma khong can thay doi code.

2. **pandas la nen tang**: Moi sem_op la mot DataFrame accessor, nghia la LOTUS mo rong pandas thay vi thay the no.

3. **faiss-cpu la bat buoc**: Mac du co WeaviateVS va QdrantVS, faiss-cpu la core dependency vi FaissVS la vector store mac dinh va sem_cluster_by dung FAISS k-means truc tiep.

4. **sentence-transformers la bat buoc**: Du co LiteLLMRM va ColBERTv2RM, sentence-transformers van la core dependency vi SentenceTransformersRM la RM mac dinh.

5. **Cascade pattern**: sem_filter, sem_join, sem_topk ho tro cascade optimization -- dung helper_lm (re/nhanh) truoc, chi goi lm chinh khi can, giam chi phi.
