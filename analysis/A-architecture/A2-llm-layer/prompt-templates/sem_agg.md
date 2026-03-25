# Prompt Template: sem_agg

## Hai loại template

`sem_agg` sử dụng 2 templates khác nhau tùy vào vị trí trong aggregation tree:
1. **Leaf template**: Cho documents gốc (level 0)
2. **Node template**: Cho intermediate summaries (level 1+)

## Verbatim Leaf Template

`_get_leaf_instruction_template` tại `sem_agg.py:12`:

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

Verbatim output (với `user_instruction="Summarize the key points"`):
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

`{{docs_str}}` được replace bởi nội dung documents (sem_agg.py:188, 206).

## Verbatim Node Template

`_get_node_instruction_template` tại `sem_agg.py:34`:

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

**Khác biệt chính so với leaf:**
- "from multiple sources" thay vì "from multiple documents"
- Thêm: "each source may be formatted differently"
- Thêm: "sources may provide opposing viewpoints or complementary information"
- Thêm: "draw connections between sources"

## Document Formatting

### Leaf documents (sem_agg.py:110-121):
```python
def leaf_doc_formatter(doc: str, ctr: int) -> str:
    return f"\n\tDocument {ctr}: {doc}"
```

Ví dụ: `\n\tDocument 1: [Title]: <<Machine Learning 101>>`

### Node summaries (sem_agg.py:123-134):
```python
def node_doc_formatter(doc: str, ctr: int) -> str:
    return f"\n\tSource {ctr}: {doc}"
```

Ví dụ: `\n\tSource 1: Machine learning covers supervised and unsupervised...`

## Hierarchical Aggregation Algorithm

sem_agg sử dụng cây (tree) để aggregate (sem_agg.py:161-219):

```
Level 0 (Leaf):  [Doc1] [Doc2] [Doc3] [Doc4] [Doc5] [Doc6]
                   \_____|_______/       \_____|_______/
Level 1 (Node):    [Summary A]           [Summary B]
                        \________________/
Level 2 (Node):          [Final Answer]
```

1. **Bắt đầu**: Tất cả documents ở level 0
2. **Grouping**: Documents được nhóm theo:
   - `partition_ids`: Documents cùng partition được aggregate cùng
   - Context window: Khi tổng tokens vượt `max_ctx_len - max_tokens`, đóng prompt hiện tại và bắt đầu prompt mới (sem_agg.py:183-184)
3. **Template selection**: Level 0 dùng leaf template, level 1+ dùng node template (sem_agg.py:170-173)
4. **Lặp**: Summaries trở thành documents mới cho level tiếp theo (sem_agg.py:217)
5. **Dừng**: Khi chỉ còn 1 summary (sem_agg.py:164)

## Message Format

Không dùng system/user message format. Chỉ có 1 user message (sem_agg.py:190, 208):
```python
batch.append([{"role": "user", "content": prompt}])
```

Prompt là template với `{{docs_str}}` được replace bởi formatted documents.

## Phân tích

1. **Không có few-shot**: `sem_agg` không hỗ trợ examples hay few-shot learning.
2. **Không có CoT**: Không hỗ trợ reasoning strategies.
3. **DSPy-style format**: Template dùng format "Follow the following format" giống DSPy framework.
4. **Context window management**: Thuật toán tự động chia documents khi vượt context window, đảm bảo mỗi LLM call nằm trong giới hạn tokens.
5. **Partition-aware**: Documents có cùng `_lotus_partition_id` được aggregate cùng, cho phép nhóm logic trước aggregation.
