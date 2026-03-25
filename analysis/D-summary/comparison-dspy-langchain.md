# So sanh LOTUS voi DSPy, LangChain, LlamaIndex

## Bang so sanh tong quan

| Khia canh | LOTUS | DSPy | LangChain | LlamaIndex |
|-----------|-------|------|-----------|------------|
| **Triet ly** | Semantic operators tren DataFrame — "SQL cho LLM" | Programmatic optimization cua LLM pipelines — "PyTorch cho LLM" | Framework ket noi LLM voi tools/data — "Swiss Army Knife" | Framework cho knowledge retrieval — "Search Engine cho LLM" |
| **Data model** | pandas DataFrame la first-class citizen. Tat ca operators nhan va tra DataFrame. `settings.py:8-14`, moi operator la DataFrame accessor | Signatures va Modules. Data la dictionaries chay qua pipeline | Documents, Messages, Chains. Data model phan tan | Documents, Nodes, Indexes. Hierarchical document model |
| **Operator model** | Tap hop operators co dinh (sem_filter, sem_map, sem_join, sem_topk, sem_agg, sem_extract, ...) dang ky qua `@pd.api.extensions.register_dataframe_accessor` — `sem_filter.py:225` | Modules (ChainOfThought, ReAct, ProgramOfThought). User define Signatures, DSPy optimize prompts | Chains, Agents, Tools. Composable nhung khong co standard operator set | Query engines, Retrievers, Response synthesizers. Focus vao retrieval pipeline |
| **Optimization** | Model cascading voi statistical guarantees (`cascade_utils.py:42-144`), Join optimizer SF vs MSF (`sem_join.py:417`), operator/LM caching (`cache.py:33`) | Teleprompter/Optimizer: automatic prompt optimization, few-shot selection, bootstrapping. MIPRO, BootstrapFewShot | Khong co built-in optimization. User tu optimize prompts | Khong co systematic optimization. Co response synthesizer strategies |
| **Accuracy guarantees** | Co: Hoeffding bounds cho recall/precision (`cascade_utils.py:52-56`), corrected targets (`cascade_utils.py:118-121`), Bonferroni correction (`cascade_utils.py:133`) | Co: metric-driven optimization. Optimizer chon prompts maximize metric tren training set | Khong co formal guarantees | Khong co formal guarantees |
| **Knowledge construction** | `sem_index` tao vector index (`sem_index.py`), `data_connectors/connectors.py` cho data sources, `web_search.py` cho web | Retrieval modules (ColBERTv2, Pinecone). Knowledge tich hop qua Retrieve module | Document loaders (100+), text splitters, vector stores. Rong nhat | Document loaders, node parsers, index types (vector, keyword, tree, knowledge graph). Sau nhat |
| **Structured + Unstructured** | Core strength: Langex (`nl_expression.py:4-8`) bridge structured columns voi unstructured LLM reasoning. `df.sem_filter("{col} condition")` | Pydantic output schemas. TypedPredictor cho structured output | Output parsers, Pydantic models. Structured output la add-on | Pydantic output, query engine metadata filters. Co nhung khong la focus |

---

## So sanh Chi tiet

### 1. Triet ly Thiet ke

**LOTUS:**
- Xay dung tren nhan thuc: du lieu enterprise nam trong tables (DataFrames)
- Moi operator la mot phep bien doi relational duoc mo rong bang LLM
- User nghĩ ve data nhu SQL, khong phai nhu LLM prompts
- Reference: `__init__.py:8-22` — tat ca operators la sem_ops imports

**DSPy:**
- Xay dung tren nhan thuc: prompts la "hyperparameters" can optimize
- User define "what" (Signatures), DSPy optimize "how" (prompts, demos)
- Focus vao pipeline optimization, khong phai data manipulation

**LangChain:**
- Xay dung tren nhan thuc: LLM can connect voi nhieu tools va data sources
- Framework lam "glue" giua components
- Focus vao flexibility va ecosystem rong

**LlamaIndex:**
- Xay dung tren nhan thuc: LLM can efficient access den knowledge
- Focus vao indexing, retrieval, va response synthesis
- Document-centric thay vi table-centric

### 2. Data Model

**LOTUS — DataFrame-centric:**
```python
# Moi operator nhan DataFrame, tra ve DataFrame
df.sem_filter("...").sem_map("...").sem_topk("...", K=5)
# Types: ImageDtype cho multimodal — image.py:12-35
# Attrs: index metadata — sem_index dung df.attrs["index_dirs"]
```

**DSPy — Module-centric:**
```python
# Data la inputs/outputs cua modules
class MySignature(dspy.Signature):
    question = dspy.InputField()
    answer = dspy.OutputField()
```

**LangChain — Chain-centric:**
```python
# Data flow qua chains
chain = prompt | llm | output_parser
result = chain.invoke({"input": "..."})
```

**LlamaIndex — Index-centric:**
```python
# Data duoc index truoc, query sau
index = VectorStoreIndex.from_documents(documents)
query_engine = index.as_query_engine()
response = query_engine.query("...")
```

### 3. Optimization Approach

**LOTUS:**
- **Cascade thresholds** tu dong hoc tu data — `cascade_utils.py:42-144`
- **Join optimizer** so sanh strategies — `sem_join.py:417-527`
- **Batching** qua litellm — `lm.py:250-252`
- **Caching** 2 tang — `cache.py:33` (operator), `lm.py:136` (LM response)
- **Khong optimize prompts** — prompts la fixed templates

**DSPy:**
- **Prompt optimization** — tu dong tim prompts tot nhat
- **Few-shot selection** — chon examples tu training set
- **Bootstrapping** — tu dong generate training data
- **Khong co data-level optimization** (batching, caching, cascading)

**LangChain:**
- **Khong co automatic optimization**
- User tu chon models, prompts, tools
- Co caching (LLM cache) nhung khong co cascading

**LlamaIndex:**
- **Index optimization** — nhieu loai index (vector, tree, keyword)
- **Response modes** — refine, compact, tree_summarize
- Khong co prompt optimization hay cascading

### 4. Accuracy Guarantees

**LOTUS — Statistical guarantees:**
```python
# cascade_utils.py:52-56 — Hoeffding bounds
def UB(mean, std_dev, s, delta):
    return mean + (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5)
```
- Recall guarantee: `P(actual_recall >= target) >= 1 - delta`
- Precision guarantee: `P(actual_precision >= target) >= 1 - delta`
- Configurable: `CascadeArgs(recall_target=0.9, precision_target=0.9, failure_probability=0.1)` — `types.py:155-159`

**DSPy — Metric-driven:**
- Optimize tren user-defined metric
- Khong co formal probability bounds
- Validation tren held-out set

**LangChain/LlamaIndex:**
- Khong co formal guarantees
- Evaluation qua external tools (RAGAS, etc.)

### 5. Unique Strengths

| Framework | Unique Strength |
|-----------|----------------|
| **LOTUS** | Semantic relational algebra tren DataFrames voi statistical guarantees. Join optimizer. Model cascading. |
| **DSPy** | Automatic prompt optimization. Programmatic LLM pipeline design. |
| **LangChain** | Ecosystem rong nhat. 100+ integrations. Agent framework. |
| **LlamaIndex** | Document indexing sau nhat. Knowledge graph integration. Multi-modal retrieval. |

### 6. Khi nao Dung Framework nao?

| Use case | Nen dung |
|----------|---------|
| Analytics tren structured data voi LLM | **LOTUS** — native DataFrame support |
| Filter/join/aggregate data voi semantic conditions | **LOTUS** — operators thiet ke cho viec nay |
| Optimize LLM prompts tu dong | **DSPy** — core strength |
| Build chatbot/agent phuc tap | **LangChain** — agent framework |
| RAG tren document collection | **LlamaIndex** — indexing va retrieval |
| Minimize LLM costs voi accuracy guarantees | **LOTUS** — cascading voi statistical bounds |
| Prototype nhanh voi nhieu LLM providers | **LangChain** — ecosystem rong |
| Research voi reproducible results | **DSPy** — deterministic optimization |

---

## Ket luan

LOTUS khac biet co ban o cho **no la query engine cho structured data**, khong phai framework cho unstructured LLM workflows.
Trong khi DSPy optimize prompts, LangChain ket noi tools, va LlamaIndex index documents, LOTUS mang semantic reasoning vao trong relational data model — noi ma phan lon enterprise data nam.

Diem doc dao nhat la **su ket hop giua SQL-like operators, model cascading voi statistical guarantees, va pandas integration** — khong framework nao khac lam duoc day du ca ba dieu nay.
