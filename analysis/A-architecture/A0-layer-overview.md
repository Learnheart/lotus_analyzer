# A0 - LOTUS Multi-Layer Architecture Overview

## Tong quan kien truc (Architecture Overview)

LOTUS duoc xay dung theo kien truc phan tang (layered architecture), moi tang co trach nhiem rieng biet.
Du lieu chay tu tren xuong (user API -> template -> LLM/embedding -> data) va ket qua tra nguoc lai.

## ASCII Architecture Diagram

```
+==============================================================================+
|                          USER CODE                                            |
|   df.sem_filter("the {title} is about AI")                                    |
|   df.sem_map("summarize {content}")                                           |
|   df.sem_search("title", "AI", K=5)                                          |
+==============================================================================+
        |                                           |
        v                                           v
+-------------------------------+   +-------------------------------+
|     API LAYER (Accessors)     |   |     API LAYER (Accessors)     |
|  LLM-based operators          |   |  Embedding-based operators    |
|-------------------------------|   |-------------------------------|
| SemFilterDataframe            |   | SemSearchDataframe            |
|   sem_filter.py:225           |   |   sem_search.py:10            |
| SemMapDataframe               |   | SemSimJoinDataframe           |
|   sem_map.py:121              |   |   sem_sim_join.py:12          |
| SemJoinDataframe              |   | SemDedupByDataframe           |
|   sem_join.py:606             |   |   sem_dedup.py:10             |
| SemAggDataframe               |   | SemIndexDataframe             |
|   sem_agg.py:226              |   |   sem_index.py:9              |
| SemTopKDataframe              |   | LoadSemIndexDataframe         |
|   sem_topk.py:624             |   |   load_sem_index.py:6         |
| SemExtractDataFrame           |   | SemClusterByDataframe         |
|   sem_extract.py:111          |   |   sem_cluster_by.py:10        |
| LLMAsJudgeDataframe           |   | SemPartitionByDataframe       |
|   llm_as_judge.py:117         |   |   sem_partition_by.py:8       |
| PairwiseJudgeDataframe        |   +-------------------------------+
|   pairwise_judge.py:13        |               |
+-------------------------------+               |
        |                                       |
        v                                       |
+-------------------------------+               |
|  TEMPLATE / NL LAYER          |               |
|-------------------------------|               |
| nl_expression.py:4            |               |
|   parse_cols() - extract      |               |
|   {col} from user instruction |               |
|   nle2str() - format cols     |               |
| task_instructions.py          |               |
|   filter_formatter :87        |               |
|   map_formatter    :213       |               |
|   extract_formatter:257       |               |
|   df2text          :325       |               |
|   df2multimodal_info :364     |               |
|   context_formatter  :40      |               |
|   user_message_formatter :68  |               |
+-------------------------------+               |
        |                                       |
        v                                       v
+-------------------------------+   +-------------------------------+
|       LLM LAYER               |   |   EMBEDDING / RETRIEVAL LAYER |
|-------------------------------|   |-------------------------------|
| LM class  (lm.py:41)         |   | RM (rm.py:10) - abstract      |
|   __call__  :123              |   |   __call__ -> _embed          |
|   batch_completion (litellm)  |   |   convert_query_to_query_vec  |
|   _process_with_rate_limiting |   |                               |
|     :258                      |   | SentenceTransformersRM        |
|   _process_with_tpm_limiting  |   |   sentence_transformers_rm    |
|     :311                      |   |   .py:11                      |
|   _update_stats :451          |   | LiteLLMRM                     |
|   _hash_messages :407         |   |   litellm_rm.py:11            |
|   cache check :136            |   | ColBERTv2RM                   |
|                               |   |   colbertv2_rm.py:17          |
| postprocessors.py             |   |                               |
|   filter_postprocess :182     |   | VS (vs.py:10) - abstract      |
|   map_postprocess    :123     |   |   index(), load_index()       |
|   extract_postprocess:149     |   |   __call__() -> RMOutput      |
|                               |   | FaissVS (faiss_vs.py:13)      |
| Cache (cache.py)              |   |   faiss.index_factory         |
|   InMemoryCache :247          |   |   faiss.write_index           |
|   SQLiteCache   :168          |   |   Flat + METRIC_INNER_PRODUCT |
|   operator_cache:33           |   +-------------------------------+
+-------------------------------+               |
        |                                       |
        v                                       v
+==============================================================================+
|                          DATA LAYER                                           |
|------------------------------------------------------------------------------|
|  pandas DataFrame                                                             |
|    attrs["index_dirs"] -> dict[col_name, index_dir]                          |
|    _lotus_partition_id column (from sem_partition_by)                         |
|    cluster_id column (from sem_cluster_by)                                   |
|                                                                              |
|  ImageDtype extension (dtype_extensions/image.py)                            |
|    get_image(i, "base64") -> base64 encoded image                            |
|                                                                              |
|  SerializationFormat (types.py:206)                                          |
|    DEFAULT: "[Col]: <<value>>\n"                                              |
|    JSON: pandas to_json                                                       |
|    XML: pandas to_xml                                                         |
|                                                                              |
|  Settings singleton (settings.py:8)                                           |
|    lm, rm, helper_lm, reranker, vs                                           |
|    enable_cache, serialization_format                                         |
+==============================================================================+
```

## Data Flow: LLM-based Operator (e.g. sem_filter)

Luong du lieu cho LLM operator di theo huong sau:

```
User call: df.sem_filter("the {title} is about AI")
    |
    v
[1] SemFilterDataframe.__call__ (sem_filter.py:334)
    |-- parse_cols("{title}") -> ["title"]               (nl_expression.py:4)
    |-- nle2str() -> "the Title is about AI"             (nl_expression.py:17)
    |-- df2multimodal_info(df, ["title"]) -> docs        (task_instructions.py:364)
    |
    v
[2] sem_filter(docs, lm, instruction) (sem_filter.py:24)
    |-- filter_formatter(model, doc, instr) -> messages  (task_instructions.py:87)
    |
    v
[3] LM.__call__(messages) (lm.py:123)
    |-- _hash_messages -> cache check                    (lm.py:407)
    |-- batch_completion (litellm)                       (lm.py:250)
    |-- _update_stats(response)                          (lm.py:451)
    |
    v
[4] filter_postprocess(outputs) (postprocessors.py:182)
    |-- parse "True"/"False" from LLM output
    |-- default fallback if unparseable
    |
    v
[5] Return filtered DataFrame
```

## Data Flow: Embedding-based Operator (e.g. sem_search)

```
User call: df.sem_search("title", "AI", K=2)
    |
    v
[1] SemSearchDataframe.__call__ (sem_search.py:92)
    |-- load index from attrs["index_dirs"]["title"]
    |
    v
[2] rm.convert_query_to_query_vector("AI")               (rm.py:53)
    |-- RM._embed(["AI"]) -> query_vectors
    |
    v
[3] vs(query_vectors, K=2)                                (faiss_vs.py:43)
    |-- faiss_index.search(query_vectors, K)
    |-- Return RMOutput(distances, indices)
    |
    v
[4] Post-filter by df_idxs (chi giu rows con ton tai trong df)
    |
    v
[5] Optional reranking via lotus.settings.reranker
    |
    v
[6] Return filtered DataFrame with top-K rows
```

## Quan he giua cac layer

| From Layer | To Layer | Mechanism | Key Interface |
|---|---|---|---|
| API -> Template | Function call | `filter_formatter()`, `map_formatter()`, `df2multimodal_info()` |
| API -> LLM | `LM.__call__()` | messages list -> `LMOutput` |
| API -> Embedding | `RM.__call__()`, `VS.__call__()` | docs -> embeddings -> `RMOutput` |
| Template -> LLM | Messages format | `list[dict[str, str]]` (role/content) |
| LLM -> Data | Return | `LMOutput.outputs` -> postprocess -> DataFrame columns |
| Embedding -> Data | Return | `RMOutput.indices` -> DataFrame row selection |
| Data -> API | `@pd.api.extensions.register_dataframe_accessor` | `self._obj` (pandas DataFrame) |

## Nhan xet kien truc (Architecture Notes)

1. **Decoupling qua litellm**: LM layer khong truc tiep goi OpenAI/Anthropic API ma thong qua `litellm.batch_completion` (lm.py:250), cho phep doi provider bang chuoi model string.

2. **Singleton Settings**: `lotus.settings` (settings.py:35) la singleton global, tat ca operator truy cap `lotus.settings.lm`, `lotus.settings.rm` de lay model instance.

3. **Operator cache la cross-cutting concern**: `@operator_cache` decorator (cache.py:33) duoc ap dung tren `__call__` cua moi accessor, cache toan bo ket qua operator dua tren hash cua DataFrame + arguments.

4. **Template layer tach biet**: Prompt formatting hoan toan nam trong `task_instructions.py`, giup thay doi prompt ma khong anh huong logic operator.
