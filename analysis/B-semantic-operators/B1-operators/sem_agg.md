# SEM_AGG — `sem_agg`

## Metadata
- **File**: `lotus/sem_ops/sem_agg.py`
- **Accessor line**: 226 (`@pd.api.extensions.register_dataframe_accessor("sem_agg")`)
- **Core function line**: 60 (`def sem_agg(...)`)
- **Type**: Unary (many-to-one)
- **Requires**: `lotus.settings.lm` (sem_agg.py:364-367)

## 1. Purpose & Use Cases

Tong hop nhieu rows thanh mot ket qua duy nhat. Su dung hierarchical tree: leaf nodes xu ly documents goc, intermediate nodes gop cac summaries lai.

**Use cases:**
- Summarization: `df.sem_agg("Summarize the key points", all_cols=True)`
- Grouped aggregation: `df.sem_agg("Summarize {journal}", group_by=["date"])`
- Question answering: `df.sem_agg("What are the main themes in {text}?")`

## 2. Call Stack Trace

```
1. SemAggDataframe.__call__()                           # sem_agg.py:354
2.   [Neu all_cols]: col_li = list(df.columns)          # sem_agg.py:370-371
2b.  [Else]: parse_cols(user_instruction)               # sem_agg.py:373
3.   Column validation                                  # sem_agg.py:377-379
4.   [Neu group_by]:
4a.    df.groupby(group_by)                              # sem_agg.py:382
4b.    ThreadPoolExecutor → process_group() parallel     # sem_agg.py:396-399
4c.    process_group → recursive sem_agg call            # sem_agg.py:326-351
5.   [Neu _lotus_partition_id exists]:
5a.    Sort by partition_id                              # sem_agg.py:402-404
6.   nle2str(user_instruction, col_li)                  # sem_agg.py:408
7.   [Neu long_context_strategy TRUNCATE/CHUNK]:
7a.    create_chunked_documents()                        # sem_agg.py:418-419
8.   [Else]: df2text(df, col_li)                        # sem_agg.py:427
9.   sem_agg(docs, lm, ...)                             # sem_agg.py:431-438
10.  Core sem_agg() — Hierarchical tree loop:
10a.   tree_level=0: leaf_instr_template                 # sem_agg.py:170-171
10b.   tree_level>0: node_instr_template                 # sem_agg.py:172-173
10c.   Token-aware batching: check context fits          # sem_agg.py:183-185
10d.   Partition boundary check                          # sem_agg.py:184
10e.   model(batch, ...)                                 # sem_agg.py:211
10f.   Loop lai voi summaries cho den khi len=1          # sem_agg.py:164
11.  Return DataFrame voi 1 row                         # sem_agg.py:441
```

## 3. Prompt Template (COPY VERBATIM)

### Leaf instruction template (sem_agg.py:12-31):
```
Your job is to provide an answer to the user's instruction given the context below from multiple documents.
Remember that your job is to answer the user's instruction by combining all relevant information from all provided documents, into a single coherent answer.
Do NOT copy the format of the sources! Instead output your answer in a coherent, well-structured manner that best answers the user instruction.
You have limited space to provide your answer, so be concise and to the point.

---

Follow the following format.

Context: relevant facts from multiple documents

Instruction: the instruction provided by the user

Answer: Write your answer

---

Context: {{docs_str}}

Instruction:  {user_instruction}

Answer:
```

### Node instruction template (sem_agg.py:34-57):
```
Your job is to provide an answer to the user's instruction given the context below from multiple sources.
Note that each source may be formatted differently and contain information about several different documents.
Remember that your job is to answer the user's instruction by combining all relevant information from all provided sources, into a single coherent answer.
The sources may provide opposing viewpoints or complementary information.
Be sure to include information from ALL relevant sources in your answer.
Do NOT copy the format of the sources, instead output your answer in a coherent, well-structured manner that best answers the user instruction.
You have limited space to provide your answer, so be concise and to the point.
You may need to draw connections between sources to provide a complete answer.

---

Follow the following format.

Context: relevant facts from multiple sources

Instruction: the instruction provided by the user

Answer: Write your answer

---

Context: {{docs_str}}

Instruction:  {user_instruction}

Answer:
```

### Leaf document format (sem_agg.py:110-121):
```
	Document {ctr}: {doc}
```

### Node document format (sem_agg.py:123-134):
```
	Source {ctr}: {doc}
```

## 4. LLM Interaction

- **Model**: `lotus.settings.lm` (sem_agg.py:431)
- **Token-aware batching**: Dem tokens cua moi document, gop vao batch cho den khi dat `model.max_ctx_len - model.max_tokens` (sem_agg.py:183)
- **Hierarchical processing**:
  - Level 0: Gop raw documents thanh summaries
  - Level 1+: Gop summaries thanh meta-summaries
  - Loop cho den khi chi con 1 summary (sem_agg.py:164)
- **Template tokens**: Tinh template tokens rieng de dam bao khong vuot context window (sem_agg.py:174)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Token-aware batching trong tree (sem_agg.py:183-185) |
| Caching | Yes | `@operator_cache` (sem_agg.py:353) |
| Cascading | No | |
| Early-termination | No | |
| Sampling | No | |
| Safe mode | Partial | Chua implement (sem_agg.py:151-153) |
| Parallel group_by | Yes | ThreadPoolExecutor (sem_agg.py:398) |
| Long context | Yes | TRUNCATE va CHUNK strategies (sem_agg.py:362, 413-419) |

### Hierarchical Tree chi tiet:
1. **Token counting**: `model.count_tokens(formatted_doc)` cho moi document (sem_agg.py:181)
2. **Batch splitting**: Khi `new_tokens + context_tokens + template_tokens > max_ctx_len - max_tokens` thi dong batch hien tai (sem_agg.py:183)
3. **Partition awareness**: Khi `partition_id != cur_partition_id and not do_fold` thi dong batch (sem_agg.py:184-185). `do_fold` = True khi moi document co partition_id khac nhau (sem_agg.py:166)

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Langex instruction (sem_agg.py:355)
- `all_cols: bool` — Neu True, dung tat ca columns (sem_agg.py:357, default=False)
- `suffix: str` — Ten column output, default `"_output"` (sem_agg.py:358)
- `group_by: list[str]` — Columns de group by (sem_agg.py:359)
- `long_context_strategy: LongContextStrategy` — TRUNCATE hoac CHUNK (sem_agg.py:362, default=CHUNK)

### Output:
- DataFrame voi 1 row (hoac 1 row per group) va column `suffix` (sem_agg.py:441)
- Neu group_by: concat tat ca group results + group_by columns (sem_agg.py:350-351, 399)

## 7. Edge Cases

1. **Column khong ton tai**: Raise `ValueError` (sem_agg.py:378-379)
2. **LM chua configure**: Raise `ValueError` (sem_agg.py:364-367)
3. **Single document**: Van chay qua tree, output 1 summary (sem_agg.py:164, 205)
4. **Partition boundary**: Neu _lotus_partition_id ton tai, sort va tach batch theo partition (sem_agg.py:402-404)
5. **Empty group**: Khi group_by co group rong, van chay nhung output rong
6. **Very long documents**: Token-aware batching tu dong tach, nhung 1 doc dai hon context window se bi TRUNCATE hoac CHUNK (sem_agg.py:413-419)

## 8. Code Examples

```python
# Basic aggregation
df.sem_agg("Summarize the key points", all_cols=True)

# Grouped aggregation
df.sem_agg("Summarize the {journal}", group_by=["date"])

# Voi partition (sau sem_partition_by)
df = df.sem_index("text", "text_idx") \
       .sem_partition_by(lotus.utils.cluster("text", 3))
df.sem_agg("Summarize {text}")

# Voi long context strategy
from lotus.types import LongContextStrategy
df.sem_agg("Summarize all {text}", long_context_strategy=LongContextStrategy.CHUNK)
```

## 9. Assessment

### Diem manh:
- **Hierarchical tree**: Xu ly so luong document lon bang cach gop dan, khong bi gioi han boi context window
- **Token-aware batching**: Tinh chinh xac token count, dam bao khong vuot context
- **Partition integration**: Tuong thich voi sem_partition_by de toi uu chat luong tong hop
- **Parallel group_by**: ThreadPoolExecutor cho grouped aggregation

### Diem yeu:
- **Safe mode chua implement**: TODO tai sem_agg.py:151-153
- **Information loss**: Hierarchical approach co the mat thong tin chi tiet qua moi level
- **No streaming**: Khong ho tro streaming output cho long aggregations
- **Validate weak**: `_validate()` la `pass` (sem_agg.py:323) — khong kiem tra gi ca
- **do_fold logic**: `do_fold = len(partition_ids) == len(set(partition_ids))` (sem_agg.py:166) — kho hieu, chi True khi moi doc co partition_id khac nhau
