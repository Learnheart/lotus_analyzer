# E3 - Aggregation and Synthesis

## Tổng quan

`sem_agg` implement hierarchical tree aggregation - progressive summarization từ leaf documents lên root summary. Đây là cơ chế chính để tổng hợp thông tin từ nhiều documents.

---

## 1. sem_agg Function

**Location**: `sem_ops/sem_agg.py:60-223`

```python
def sem_agg(docs, model, user_instruction, partition_ids, ...):
```

---

## 2. Hierarchical Tree Structure

### Leaf Template (`sem_agg.py:12-31`)
```python
def _get_leaf_instruction_template(user_instruction):
    return (
        "Your job is to provide an answer to the user's instruction "
        "given the context below from multiple documents.\n"
        "Remember that your job is to answer the user's instruction "
        "by combining all relevant information from all provided documents...\n"
        "Context: {{docs_str}}\n\n"
        f"Instruction:  {user_instruction}\n\nAnswer:\n"
    )
```

### Node Template (`sem_agg.py:34-57`)
```python
def _get_node_instruction_template(user_instruction):
    return (
        "Your job is to provide an answer to the user's instruction "
        "given the context below from multiple sources.\n"
        "Note that each source may be formatted differently...\n"
        "Be sure to include information from ALL relevant sources...\n"
        "Context: {{docs_str}}\n\n"
        f"Instruction:  {user_instruction}\n\nAnswer:\n"
    )
```

Sự khác biệt:
- **Leaf**: "multiple documents" - xử lý raw documents
- **Node**: "multiple sources" - xử lý previous summaries, nhấn mạnh "opposing viewpoints" và "complementary information"

### Document Formatters
```python
def leaf_doc_formatter(doc, ctr):
    return f"\n\tDocument {ctr}: {doc}"      # sem_agg.py:121

def node_doc_formatter(doc, ctr):
    return f"\n\tSource {ctr}: {doc}"         # sem_agg.py:134
```

---

## 3. Token Budget Management

**Location**: `sem_agg.py:174-209`

```python
template_tokens = model.count_tokens(template)                    # :174
context_tokens = 0                                                 # :175

for idx in range(len(doc_list)):
    formatted_doc = doc_formatter(tree_level, doc_list[idx], doc_ctr)
    new_tokens = model.count_tokens(formatted_doc)                 # :181

    if (new_tokens + context_tokens + template_tokens >
        model.max_ctx_len - model.max_tokens):                     # :183
        # Close current prompt, start new batch
        prompt = template.replace("{{docs_str}}", context_str)     # :188
        batch.append([{"role": "user", "content": prompt}])        # :190
```

Quá trình:
1. Tính token budget: `max_ctx_len - max_tokens - template_tokens`
2. Thêm documents vào context cho đến khi hết budget (`sem_agg.py:183`)
3. Khi hết budget → close current prompt, bắt đầu prompt mới
4. Partition boundary: nếu `partition_id` thay đổi, cũng close prompt (`sem_agg.py:184`)

---

## 4. Tree Aggregation Loop

**Location**: `sem_agg.py:164-219`

```python
tree_level = 0
while len(doc_list) != 1 or summaries == []:    # :164
    # ... build batches ...
    lm_output = model(batch, ...)                # :211
    summaries = lm_output.outputs                 # :213
    doc_list = summaries                          # :217
    tree_level += 1                               # :219
```

Ví dụ với 100 documents, context window fits 10:
```
Level 0 (leaf):  100 docs → 10 batches → 10 summaries
Level 1 (node):  10 summaries → 1 batch → 1 final answer
```

Nếu summaries vẫn quá nhiều:
```
Level 0: 1000 docs → 100 summaries
Level 1: 100 summaries → 10 summaries
Level 2: 10 summaries → 1 final answer
```

---

## 5. Partition-aware Aggregation

Khi `partition_ids` khác nhau, documents cùng partition được nhóm lại:

```python
partition_id = partition_ids[idx]                           # :179
if partition_id != cur_partition_id and not do_fold:        # :184-185
    # Close current prompt for this partition
```

`do_fold` (`sem_agg.py:166`):
```python
do_fold = len(partition_ids) == len(set(partition_ids))
```
Khi mỗi document có partition_id riêng (all unique), cho phép fold across partitions.

---

## 6. Group-by Parallelism

**Location**: `sem_agg.py:396-399`

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

Mỗi group được aggregate song song.

---

## 7. Long Context Strategy Integration

**Location**: `sem_agg.py:411-428`

```python
if long_context_strategy in (LongContextStrategy.TRUNCATE, LongContextStrategy.CHUNK):
    docs_input = create_chunked_documents(
        self._obj, col_li, lotus.settings.lm, long_context_strategy, template_tokens  # :418-419
    )
```

Default: `LongContextStrategy.CHUNK` (`sem_agg.py:362`)

---

## 8. Kết luận

sem_agg's hierarchical aggregation:
- Two-level templates (leaf for docs, node for summaries)
- Token-budget-aware batching
- Progressive summarization tree
- Partition-aware grouping
- Parallel group-by processing
- Integrated long context handling
