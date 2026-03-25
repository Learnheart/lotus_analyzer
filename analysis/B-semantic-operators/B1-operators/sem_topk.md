# SEM_TOPK — `sem_topk`

## Metadata
- **File**: `lotus/sem_ops/sem_topk.py`
- **Accessor line**: 624 (`@pd.api.extensions.register_dataframe_accessor("sem_topk")`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_topk.py:747-751)
- **Sorting methods**: quick (347), heap (560), naive (276), quick-sem (782)

## 1. Purpose & Use Cases

Sap xep DataFrame theo tieu chi ngu nghia va tra ve K rows tot nhat. Su dung pairwise comparison qua LLM.

**Use cases:**
- Best match: `df.sem_topk("The {title} is best for beginners", K=3)`
- Ranking: `df.sem_topk("The {product} has the best value", K=5)`
- Selection: `df.sem_topk("The {candidate} is most qualified", K=1)`

## 2. Call Stack Trace

```
1. SemTopKDataframe.__call__()                          # sem_topk.py:735
2.   parse_cols(user_instruction)                       # sem_topk.py:754
3.   Column validation                                  # sem_topk.py:758-760
4.   [Neu group_by]:
4a.    ThreadPoolExecutor → process_group() parallel    # sem_topk.py:770-773
5.   [Neu method="quick-sem"]:
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

- **Binary comparison**: Moi LLM call so sanh dung 2 documents (sem_topk.py:16-80)
- **Batch comparison**: `compare_batch_binary()` gui nhieu pairs cung luc (sem_topk.py:132-173)
- **Parse result**: `parse_ans_binary()` trich "Document 1" hoac "Document 2" tu output (sem_topk.py:83-129)
  - Regex: `r"Document[\s*](\d+)"` (sem_topk.py:119)
  - Fallback: `r"(\d+)"` (sem_topk.py:121)
  - Default: True (doc1 wins) khi parse that bai (sem_topk.py:125-128)
- **DeepSeek support**: Them `<think></think>` tag instructions (sem_topk.py:72-76)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | compare_batch_binary gui nhieu pairs 1 lan (sem_topk.py:169) |
| Caching | Yes | `@operator_cache` (sem_topk.py:734) |
| Cascading | Yes | compare_batch_binary_cascade voi helper_lm (sem_topk.py:176-273) |
| Early-termination | Partial | Quicksort chi sort can thiet cho top-K (sem_topk.py:479-483) |
| Sampling | No | |
| Safe mode | Yes | Estimate calls va tokens (sem_topk.py:393-399, 597-603) |
| Parallel group_by | Yes | ThreadPoolExecutor (sem_topk.py:772) |

### Sorting Methods chi tiet:

**1. `llm_quicksort` (sem_topk.py:347-488)** — O(N log N) average, O(N log K) cho top-K:
- Partition function voi pivot selection (sem_topk.py:407-463)
- Embedding optimization: chon pivot la K-th closest element (sem_topk.py:413-416)
- Cascade support: dung helper_lm cho low-confidence comparisons (sem_topk.py:438-455)
- Recursive: chi sort partition can thiet cho top-K (sem_topk.py:479-483)

**2. `llm_heapsort` (sem_topk.py:560-621)** — O(N log N):
- Dung `HeapDoc` class voi custom `__lt__` operator (sem_topk.py:491-557)
- `__lt__` goi LLM cho moi comparison (sem_topk.py:526-557)
- `heapq.nsmallest(K, heap)` (sem_topk.py:613)
- Moi comparison = 1 LLM call (khong batch) — cham hon quicksort

**3. `llm_naive_sort` (sem_topk.py:276-344)** — O(N^2):
- So sanh tat ca pairs (N*(N-1)/2) (sem_topk.py:312-314)
- Voting system: moi doc duoc 1 vote khi thang (sem_topk.py:327-338)
- Sort theo so votes (sem_topk.py:341)

**4. `quick-sem` (sem_topk.py:782-788)**:
- Pre-sort bang embedding similarity (sem_index + sem_search)
- Sau do dung quicksort voi embedding-optimized pivot selection

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Ranking criteria (sem_topk.py:736)
- `K: int` — So rows tra ve (sem_topk.py:737)
- `method: str` — "quick", "heap", "naive", "quick-sem" (sem_topk.py:738, default="quick")
- `cascade_threshold: float | None` — Threshold cho cascade (sem_topk.py:741)
- `group_by: list[str]` — Group by columns (sem_topk.py:740)

### Output:
- DataFrame voi K rows, sorted by relevance (sem_topk.py:825-827)
- **return_explanations=True** (voi ZS_COT): Them column `explanation` (sem_topk.py:829-839)
- **return_stats=True**: Tra ve tuple (DataFrame, stats) (sem_topk.py:841-846)

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

1. **K > len(df)**: Tra ve toan bo DataFrame (sem_topk.py:827)
2. **Parse "Document N" that bai**: Default True — doc1 thang (sem_topk.py:125-128)
3. **quick-sem yeu cau 1 column**: Assert error (sem_topk.py:783)
4. **Heap method**: Moi comparison = 1 LLM call rieng le, khong batch — performance issue
5. **Cascade cho heap**: Khong ho tro — chi quicksort co cascade (sem_topk.py:401)
6. **Invalid method**: Raise `ValueError` (sem_topk.py:822-823)

## 8. Code Examples

```python
# Basic top-K
df.sem_topk("The {title} is best for beginners", K=3)

# Voi heapsort
df.sem_topk("The {product} has best value", K=5, method="heap")

# Voi embedding optimization
df = df.sem_index("title", "title_idx")
df.sem_topk("The {title} is most relevant", K=3, method="quick-sem")

# Voi cascade
df.sem_topk("The {title} is best", K=3, cascade_threshold=0.8)

# Grouped top-K
df.sem_topk("The {product} is best", K=2, group_by=["category"])

# Voi ZS-CoT explanations
df.sem_topk(
    "The {title} is most relevant",
    K=3,
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True
)
```

## 9. Assessment

### Diem manh:
- **Multiple algorithms**: 4 methods cho cac use cases khac nhau
- **Smart top-K**: Quicksort chi sort phan can thiet, tiet kiem LLM calls
- **Embedding optimization**: quick-sem dung embeddings de chon pivot tot hon
- **Cascade support**: Quicksort ho tro cascade voi helper_lm

### Diem yeu:
- **HeapDoc design**: Class variable sharing (sem_topk.py:507-511) — khong thread-safe, co the bi race condition khi dung parallel group_by
- **Parse fragility**: Default True khi parse that bai (sem_topk.py:125) — co the anh huong ket qua sorting
- **No stability guarantee**: Quicksort khong stable — equal elements co the bi dao order
- **File length**: 848 dong, nhieu methods — nen tach thanh module rieng
- **Typo**: "to to select" (sem_topk.py:53) — loi double "to"
- **return_explanations chi voi ZS_COT**: Chi co explanation khi `strategy == ReasoningStrategy.ZS_COT` (sem_topk.py:829) — han che
