# E4 - Ranking Unstructured Data

## Tổng quan

`sem_topk` cung cấp nhiều sorting algorithms để rank unstructured data dựa trên LLM pairwise comparisons. Các methods: "quick" (quicksort), "heap" (heapsort), "naive" (all-pairs), "quick-sem" (embedding-informed quicksort).

---

## 1. Method Selection

**Location**: `sem_topk.py:794-823`

```python
if method in ["quick", "quick-sem"]:
    output = llm_quicksort(docs, model, ..., K,
                           embedding=method == "quick-sem", ...)    # :795-804
elif method == "heap":
    output = llm_heapsort(docs, model, ..., K, ...)                # :806-813
elif method == "naive":
    output = llm_naive_sort(docs, model, ..., ...)                 # :815-821
```

---

## 2. Quicksort (method="quick")

**Location**: `sem_topk.py:347-488`

```python
def llm_quicksort(docs, model, user_instruction, K,
                   embedding=False, strategy=None, cascade_threshold=None, ...):
```

### Partition (`sem_topk.py:407-463`)

```python
def partition(indexes, low, high, K):
    if embedding:
        # Informed pivot selection
        pivot_value = heapq.nsmallest(K, indexes[low:high+1])[-1]  # :414
    else:
        # Random pivot
        pivot_index = np.random.randint(low, high + 1)             # :420

    pairs = [(docs[indexes[j]], pivot) for j in range(low, high)]  # :426
    comparisons, explanations, tokens = compare_batch_binary(      # :428
        pairs, model, user_instruction, ...
    )
```

- Batch all comparisons against pivot in one LLM call
- Partition based on results

### Recursive quicksort (`sem_topk.py:465-487`)
```python
def quicksort_recursive(indexes, low, high, K):
    pi = partition(indexes, low, high, K)              # :476
    left_size = pi - low
    if left_size + 1 >= K:
        quicksort_recursive(indexes, low, pi - 1, K)  # :480 - only sort left
    else:
        quicksort_recursive(indexes, low, pi - 1, left_size)   # :482
        quicksort_recursive(indexes, pi + 1, high, K - left_size - 1)  # :483
```

**Optimization**: Chỉ sort phần cần thiết cho top-K (partial quickselect).

---

## 3. Heapsort (method="heap")

**Location**: `sem_topk.py:560-621`

### HeapDoc class (`sem_topk.py:491-557`)

```python
class HeapDoc:
    num_calls: int = 0            # :507
    total_tokens: int = 0         # :508

    def __lt__(self, other):      # :526
        assert HeapDoc.model is not None
        prompt = get_match_prompt_binary(self.doc, other.doc, ...)  # :543-544
        HeapDoc.num_calls += 1                                      # :546
        result = HeapDoc.model([prompt], ...)                       # :548
        is_better, explanation = parse_ans_binary(result.outputs[0])  # :549
        return is_better                                            # :557
```

`__lt__` gọi LLM cho mỗi comparison - Python's heap operations tự động gọi khi cần.

### Heap extraction (`sem_topk.py:610-614`)
```python
HeapDoc.model = model
heap = [HeapDoc(docs[idx], user_instruction, idx) for idx in range(N)]  # :611
heap = heapq.nsmallest(K, heap)                                         # :613
indexes = [heapq.heappop(heap).idx for _ in range(len(heap))]          # :614
```

`heapq.nsmallest(K, heap)` internally builds heap và extracts K smallest.

---

## 4. Naive Sort (method="naive")

**Location**: `sem_topk.py:276-344`

```python
def llm_naive_sort(docs, model, user_instruction, ...):
    N = len(docs)
    pairs = []
    for i in range(N):
        for j in range(i + 1, N):
            pairs.append((docs[i], docs[j]))           # :313-314

    comparisons, explanations, tokens = compare_batch_binary(  # :322
        pairs, model, user_instruction, ...
    )

    votes = [0] * N                                    # :327
    for i in range(N):
        for j in range(i + 1, N):
            if comparisons[idx]:
                votes[i] += 1                          # :334
            else:
                votes[j] += 1                          # :337

    indexes = sorted(range(len(votes)),
                     key=lambda i: votes[i], reverse=True)  # :341
```

- O(N^2) comparisons: mọi pairs
- Voting: mỗi comparison cho 1 vote cho winner
- Sort by vote count

---

## 5. Quick-Sem (method="quick-sem")

**Location**: `sem_topk.py:782-788`

```python
if method == "quick-sem":
    col_name = col_li[0]
    self._obj = self._obj.sem_index(col_name, f"{col_name}_lotus_index")  # :786
    self._obj = self._obj.sem_search(col_name, user_instruction, len(self._obj))  # :787
```

Pre-processing:
1. Index column bằng embeddings
2. Sort toàn bộ DataFrame bằng embedding similarity
3. Quicksort sử dụng informed pivot selection (`sem_topk.py:413-414`):
   ```python
   pivot_value = heapq.nsmallest(K, indexes[low:high+1])[-1]
   ```

Pivot chọn dựa trên embedding ranking → giảm so sánh không cần thiết.

---

## 6. Cascade trong TopK

**Location**: `sem_topk.py:176-273`

```python
def compare_batch_binary_cascade(pairs, model, user_instruction,
                                  cascade_threshold, ...):
    helper_output = helper_lm(match_prompts, kwargs={"logprobs": True})  # :231
    # ... check confidence ...
    for idx_j in range(len(helper_tokens[idx]) - 1, -1, -1):
        if helper_tokens[idx][idx_j].strip(" \n").isnumeric():           # :249
            conf = helper_confidences[idx][idx_j]                        # :250
            if conf >= cascade_threshold:                                # :251
                high_conf_idxs.add(idx)                                  # :252
```

Two-stage: helper LM makes predictions → low-confidence sent to large LM.

---

## 7. Complexity Comparison

| Method | Best Case | Worst Case | Notes |
|---|---|---|---|
| quick | O(N log K) | O(N^2) | Partial sort, random pivot |
| quick-sem | O(N log K) | O(N^2) | Informed pivot, better average |
| heap | O(N log K) | O(N log N) | Guaranteed top-K |
| naive | O(N^2) | O(N^2) | All-pairs, most robust |

---

## 8. Kết luận

sem_topk cung cấp flexibility trong ranking:
- quicksort: balanced performance/accuracy
- heapsort: guaranteed complexity
- naive: most accurate (all pairs compared)
- quick-sem: best for embedding-aligned ranking
- cascade support: reduce LLM calls
