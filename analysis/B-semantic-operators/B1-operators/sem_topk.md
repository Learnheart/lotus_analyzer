# SEM_TOPK — `sem_topk`

## Metadata
- **File**: `lotus/sem_ops/sem_topk.py`
- **Accessor line**: 624 (`@pd.api.extensions.register_dataframe_accessor("sem_topk")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_topk.py:747-751)
- **Sorting methods**: quick (347), heap (560), naive (276), quick-sem (782)

## 1. Purpose & Use Cases

Sắp xếp DataFrame theo tiêu chí ngữ nghĩa và trả về K rows tốt nhất. Sử dụng pairwise comparison qua LLM.

**Use cases:**
- Best match: `df.sem_topk("The {title} is best for beginners", K=3)`
- Ranking: `df.sem_topk("The {product} has the best value", K=5)`
- Selection: `df.sem_topk("The {candidate} is most qualified", K=1)`

## 2. Call Stack Trace

```
1. SemTopKDataframe.__call__()                          # sem_topk.py:735
2.   parse_cols(user_instruction)                       # sem_topk.py:754
3.   Column validation                                  # sem_topk.py:758-760
4.   [Nếu group_by]:
4a.    ThreadPoolExecutor → process_group() parallel    # sem_topk.py:770-773
5.   [Nếu method="quick-sem"]:
5a.    sem_index() + sem_search() pre-sort by embedding # sem_topk.py:786-788
6.   df2multimodal_info(df, col_li)                     # sem_topk.py:790
7.   nle2str(user_instruction, col_li)                  # sem_topk.py:792
8.   Dispatch theo method:
8a.   "quick"/"quick-sem" → llm_quicksort()             # sem_topk.py:795-804
8b.   "heap" → llm_heapsort()                           # sem_topk.py:805-813
8c.   "naive" → llm_naive_sort()                        # sem_topk.py:814-820
9.   Reindex DataFrame theo sorted indexes              # sem_topk.py:825-827
10.  Return top K rows                                  # sem_topk.py:847
```

### Pairwise comparison flow:
```
get_match_prompt_binary(doc1, doc2, instruction, model)  # sem_topk.py:16-80
→ model([prompt])                                         # sem_topk.py:169
→ parse_ans_binary(answer)                                # sem_topk.py:83-129
→ True = doc1 wins, False = doc2 wins
```

## 3. Prompt Template (COPY VERBATIM)

### System instruction — standard (sem_topk.py:60-66):
```
Your job is to to select and return the most relevant document to the user's question.
Carefully read the user's question and the two documents provided below.
Respond only with the label of the document such as "Document NUMBER".
NUMBER must be either 1 or 2, depending on which document is most relevant.
You must pick a number and cannot say things like "None" or "Neither"
```

### System instruction — ZS-CoT (sem_topk.py:52-58):
```
Your job is to to select and return the most relevant document to the user's question.
Carefully read the user's question and the two documents provided below.
First give your reasoning. Then you MUST end your output with "Answer: Document 1 or Document 2"
You must pick a number and cannot say things like "None" or "Neither"
Remember to explicitly state "Answer:" at the end before your choice.
```

### User message format (sem_topk.py:67-70):
```
Question: {user_instruction}

Document 1:
{content_text_1}

Document 2:
{content_text_2}
```

## 4. LLM Interaction

- **Binary comparison**: Mỗi LLM call so sánh đúng 2 documents (sem_topk.py:16-80)
- **Batch comparison**: `compare_batch_binary()` gửi nhiều pairs cùng lúc (sem_topk.py:132-173)
- **Parse result**: `parse_ans_binary()` trích "Document 1" hoặc "Document 2" từ output (sem_topk.py:83-129)
  - Regex: `r"Document[\s*](\d+)"` (sem_topk.py:119)
  - Fallback: `r"(\d+)"` (sem_topk.py:121)
  - Default: True (doc1 wins) khi parse thất bại (sem_topk.py:125-128)
- **DeepSeek support**: Thêm `<think></think>` tag instructions (sem_topk.py:72-76)

## 5. Optimization

| Feature | Status | Chi tiết |
|---|---|---|
| Batching | Yes | compare_batch_binary gửi nhiều pairs 1 lần (sem_topk.py:169) |
| Caching | Yes | `@operator_cache` (sem_topk.py:734) |
| Cascading | Yes | compare_batch_binary_cascade với helper_lm (sem_topk.py:176-273) |
| Early-termination | Partial | Quicksort chỉ sort cần thiết cho top-K (sem_topk.py:479-483) |
| Sampling | No | |
| Safe mode | Yes | Estimate calls và tokens (sem_topk.py:393-399, 597-603) |
| Parallel group_by | Yes | ThreadPoolExecutor (sem_topk.py:772) |

### Sorting Methods chi tiết:

**1. `llm_quicksort` (sem_topk.py:347-488)** — O(N log N) average, O(N log K) cho top-K:
- Partition function với pivot selection (sem_topk.py:407-463)
- Embedding optimization: chọn pivot là K-th closest element (sem_topk.py:413-416)
- Cascade support: dùng helper_lm cho low-confidence comparisons (sem_topk.py:438-455)
- Recursive: chỉ sort partition cần thiết cho top-K (sem_topk.py:479-483)

**2. `llm_heapsort` (sem_topk.py:560-621)** — O(N log N):
- Dùng `HeapDoc` class với custom `__lt__` operator (sem_topk.py:491-557)
- `__lt__` gọi LLM cho mỗi comparison (sem_topk.py:526-557)
- `heapq.nsmallest(K, heap)` (sem_topk.py:613)
- Mỗi comparison = 1 LLM call (không batch) — chậm hơn quicksort

**3. `llm_naive_sort` (sem_topk.py:276-344)** — O(N^2):
- So sánh tất cả pairs (N*(N-1)/2) (sem_topk.py:312-314)
- Voting system: mỗi doc được 1 vote khi thắng (sem_topk.py:327-338)
- Sort theo số votes (sem_topk.py:341)

**4. `quick-sem` (sem_topk.py:782-788)**:
- Pre-sort bằng embedding similarity (sem_index + sem_search)
- Sau đó dùng quicksort với embedding-optimized pivot selection

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Ranking criteria (sem_topk.py:736)
- `K: int` — Số rows trả về (sem_topk.py:737)
- `method: str` — "quick", "heap", "naive", "quick-sem" (sem_topk.py:738, default="quick")
- `cascade_threshold: float | None` — Threshold cho cascade (sem_topk.py:741)
- `group_by: list[str]` — Group by columns (sem_topk.py:740)

### Output:
- DataFrame với K rows, sorted by relevance (sem_topk.py:825-827)
- **return_explanations=True** (với ZS_COT): Thêm column `explanation` (sem_topk.py:829-839)
- **return_stats=True**: Trả về tuple (DataFrame, stats) (sem_topk.py:841-846)

### Stats format:
```python
stats = {
    "total_tokens": int,
    "total_llm_calls": int,
    "explanations": {doc_idx: [list of explanations]},
    # Cascade only:
    "total_small_tokens": int,
    "total_large_tokens": int,
    "total_small_calls": int,
    "total_large_calls": int,
}
```

## 7. Edge Cases

1. **K > len(df)**: Trả về toàn bộ DataFrame (sem_topk.py:827)
2. **Parse "Document N" thất bại**: Default True — doc1 thắng (sem_topk.py:125-128)
3. **quick-sem yêu cầu 1 column**: Assert error (sem_topk.py:783)
4. **Heap method**: Mỗi comparison = 1 LLM call riêng lẻ, không batch — performance issue
5. **Cascade cho heap**: Không hỗ trợ — chỉ quicksort có cascade (sem_topk.py:401)
6. **Invalid method**: Raise `ValueError` (sem_topk.py:822-823)

## 8. Code Examples

```python
# Basic top-K
df.sem_topk("The {title} is best for beginners", K=3)

# Với heapsort
df.sem_topk("The {product} has best value", K=5, method="heap")

# Với embedding optimization
df = df.sem_index("title", "title_idx")
df.sem_topk("The {title} is most relevant", K=3, method="quick-sem")

# Với cascade
df.sem_topk("The {title} is best", K=3, cascade_threshold=0.8)

# Grouped top-K
df.sem_topk("The {product} is best", K=2, group_by=["category"])

# Với ZS-CoT explanations
df.sem_topk(
    "The {title} is most relevant",
    K=3,
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True
)
```

## 9. Assessment

### Điểm mạnh:
- **Multiple algorithms**: 4 methods cho các use cases khác nhau
- **Smart top-K**: Quicksort chỉ sort phần cần thiết, tiết kiệm LLM calls
- **Embedding optimization**: quick-sem dùng embeddings để chọn pivot tốt hơn
- **Cascade support**: Quicksort hỗ trợ cascade với helper_lm

### Điểm yếu:
- **HeapDoc design**: Class variable sharing (sem_topk.py:507-511) — không thread-safe, có thể bị race condition khi dùng parallel group_by
- **Parse fragility**: Default True khi parse thất bại (sem_topk.py:125) — có thể ảnh hưởng kết quả sorting
- **No stability guarantee**: Quicksort không stable — equal elements có thể bị đảo order
- **File length**: 848 dòng, nhiều methods — nên tách thành module riêng
- **Typo**: "to to select" (sem_topk.py:53) — lỗi double "to"
- **return_explanations chỉ với ZS_COT**: Chỉ có explanation khi `strategy == ReasoningStrategy.ZS_COT` (sem_topk.py:829) — hạn chế
