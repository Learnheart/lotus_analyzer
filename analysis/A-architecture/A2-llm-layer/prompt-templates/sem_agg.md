# Prompt Template: sem_agg

## Hai loai template

`sem_agg` su dung 2 templates khac nhau tuy vao vi tri trong aggregation tree:
1. **Leaf template**: Cho documents goc (level 0)
2. **Node template**: Cho intermediate summaries (level 1+)

## Verbatim Leaf Template

`_get_leaf_instruction_template` tai `sem_agg.py:12`:

```python
def _get_leaf_instruction_template(user_instruction: str) -> str:
    return (
        "Your job is to provide an answer to the user's instruction given the context below from multiple documents.\n"
        "Remember that your job is to answer the user's instruction by combining all relevant information from all provided documents, into a single coherent answer.\n"
        "Do NOT copy the format of the sources! Instead output your answer in a coherent, well-structured manner that best answers the user instruction.\n"
        "You have limited space to provide your answer, so be concise and to the point.\n\n---\n\n"
        "Follow the following format.\n\nContext: relevant facts from multiple documents\n\n"
        "Instruction: the instruction provided by the user\n\nAnswer: Write your answer\n\n---\n\n"
        "Context: {{docs_str}}\n\n"
        f"Instruction:  {user_instruction}\n\nAnswer:\n"
    )
```

Verbatim output (voi `user_instruction="Summarize the key points"`):
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

Instruction:  Summarize the key points

Answer:
```

`{{docs_str}}` duoc replace boi noi dung documents (sem_agg.py:188, 206).

## Verbatim Node Template

`_get_node_instruction_template` tai `sem_agg.py:34`:

```python
def _get_node_instruction_template(user_instruction: str) -> str:
    return (
        "Your job is to provide an answer to the user's instruction given the context below from multiple sources.\n"
        "Note that each source may be formatted differently and contain information about several different documents.\n"
        "Remember that your job is to answer the user's instruction by combining all relevant information from all provided sources, into a single coherent answer.\n"
        "The sources may provide opposing viewpoints or complementary information.\n"
        "Be sure to include information from ALL relevant sources in your answer.\n"
        "Do NOT copy the format of the sources, instead output your answer in a coherent, well-structured manner that best answers the user instruction.\n"
        "You have limited space to provide your answer, so be concise and to the point.\n"
        "You may need to draw connections between sources to provide a complete answer.\n\n---\n\n"
        "Follow the following format.\n\nContext: relevant facts from multiple sources\n\n"
        "Instruction: the instruction provided by the user\n\nAnswer: Write your answer\n\n---\n\n"
        "Context: {{docs_str}}\n\n"
        f"Instruction:  {user_instruction}\n\nAnswer:\n"
    )
```

**Khac biet chinh so voi leaf:**
- "from multiple sources" thay vi "from multiple documents"
- Them: "each source may be formatted differently"
- Them: "sources may provide opposing viewpoints or complementary information"
- Them: "draw connections between sources"

## Document Formatting

### Leaf documents (sem_agg.py:110-121):
```python
def leaf_doc_formatter(doc: str, ctr: int) -> str:
    return f"\n\tDocument {ctr}: {doc}"
```

Vi du: `\n\tDocument 1: [Title]: <<Machine Learning 101>>`

### Node summaries (sem_agg.py:123-134):
```python
def node_doc_formatter(doc: str, ctr: int) -> str:
    return f"\n\tSource {ctr}: {doc}"
```

Vi du: `\n\tSource 1: Machine learning covers supervised and unsupervised...`

## Hierarchical Aggregation Algorithm

sem_agg su dung cay (tree) de aggregate (sem_agg.py:161-219):

```
Level 0 (Leaf):  [Doc1] [Doc2] [Doc3] [Doc4] [Doc5] [Doc6]
                   \_____|_______/       \_____|_______/
Level 1 (Node):    [Summary A]           [Summary B]
                        \________________/
Level 2 (Node):          [Final Answer]
```

1. **Bat dau**: Tat ca documents o level 0
2. **Grouping**: Documents duoc nhom theo:
   - `partition_ids`: Documents cung partition duoc aggregate cung
   - Context window: Khi tong tokens vuot `max_ctx_len - max_tokens`, dong prompt hien tai va bat dau prompt moi (sem_agg.py:183-184)
3. **Template selection**: Level 0 dung leaf template, level 1+ dung node template (sem_agg.py:170-173)
4. **Lap**: Summaries tro thanh documents moi cho level tiep theo (sem_agg.py:217)
5. **Dung**: Khi chi con 1 summary (sem_agg.py:164)

## Message Format

Khong dung system/user message format. Chi co 1 user message (sem_agg.py:190, 208):
```python
batch.append([{"role": "user", "content": prompt}])
```

Prompt la template voi `{{docs_str}}` duoc replace boi formatted documents.

## Phan tich

1. **Khong co few-shot**: `sem_agg` khong ho tro examples hay few-shot learning.
2. **Khong co CoT**: Khong ho tro reasoning strategies.
3. **DSPy-style format**: Template dung format "Follow the following format" giong DSPy framework.
4. **Context window management**: Thuat toan tu dong chia documents khi vuot context window, dam bao moi LLM call nam trong gioi han tokens.
5. **Partition-aware**: Documents co cung `_lotus_partition_id` duoc aggregate cung, cho phep nhom logic truoc aggregation.
