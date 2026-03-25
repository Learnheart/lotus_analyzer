# E3 - LLM as Judge

## Tổng quan

LOTUS sử dụng LLM như "judge" trong nhiều operators - đánh giá boolean conditions, pairwise comparisons, và explicit evaluation. LLM reasoning là core mechanism cho semantic operations.

---

## 1. sem_filter: LLM as Boolean Judge

**Prompt** (`task_instructions.py:99-101`):
```
System: The user will provide a claim and some relevant context.
        Your job is to determine whether the claim is true for the given context.

User: Context: [data]
      Claim: {user_instruction}
```

LLM trả lời "True" hoặc "False". Post-processing (`postprocessors.py:200-211`):
```python
def process_outputs(answer):
    if "True" in answer:     # :205
        return True
    elif "False" in answer:  # :207
        return False
    else:
        return default       # :210 - fallback khi không parse được
```

---

## 2. sem_topk: LLM as Pairwise Comparator

**Prompt** (`sem_topk.py:51-66`):
```
System: Your job is to select and return the most relevant document to the user's question.
        Respond only with the label of the document such as "Document NUMBER".
        NUMBER must be either 1 or 2.

User: Question: {user_instruction}
      Document 1: [doc1 content]
      Document 2: [doc2 content]
```

LLM trả lời "Document 1" hoặc "Document 2".

### parse_ans_binary (`sem_topk.py:83-129`)
```python
matches = list(re.finditer(r"Document[\s*](\d+)", answer, re.IGNORECASE))  # :119
if len(matches) == 0:
    matches = list(re.finditer(r"(\d+)", answer, re.IGNORECASE))           # :121
ans = int(matches[-1].group(1)) - 1                                        # :122
return ans == 0, cot_explanation                                            # :126
```

Fallback: nếu không tìm được "Document X", tìm bất kỳ số nào. Default: True (Document 1) (`sem_topk.py:125`).

---

## 3. sem_join: Reuses sem_filter

`sem_join` internally gọi `sem_filter` cho mỗi pair (`sem_join.py:137-147`):
```python
output = sem_filter(
    all_docs,                      # merged (left, right) documents
    model,
    user_instruction,
    ...
)
```

Mỗi pair (l1[i], l2[j]) được đánh giá True/False bởi LLM.

---

## 4. llm_as_judge Eval

**Location**: `evals/llm_as_judge.py:16-114`

Explicit judge evaluation framework:

```python
def llm_as_judge(docs, model, judge_instruction,
                 response_format=None, n_trials=1, ...):
```

### Đặc điểm
- **n_trials**: Chạy evaluation nhiều lần (`llm_as_judge.py:22`) - parallel qua ThreadPoolExecutor (`llm_as_judge.py:83`)
- **Custom system prompt**: Default là "You are an intelligent, rigorous, and fair evaluator" (`llm_as_judge.py:70-74`)
- **Response format**: Hỗ trợ Pydantic BaseModel cho structured output (`llm_as_judge.py:21`)
- **Cache disabled**: `lotus.settings.enable_cache = False` (`llm_as_judge.py:82`) để đảm bảo independent trials
- Internally dùng `sem_map` cho evaluation (`llm_as_judge.py:86-100`)

---

## 5. pairwise_judge

**Location**: `evals/pairwise_judge.py:13-158`

So sánh hai columns side-by-side:

```python
class PairwiseJudgeDataframe:
    def __call__(self, col1, col2, judge_instruction,
                 n_trials=1, permute_cols=False, ...):
```

### permute_cols cho Position Bias Mitigation
Khi `permute_cols=True` (`pairwise_judge.py:100-140`):
```python
if permute_cols:
    if n_trials % 2:
        raise ValueError("Number of trials should be even")  # :101-102
    for c1, c2 in [(col1, col2), (col2, col1)]:              # :105-107
        output = self._obj.pairwise_judge(
            col1=c1, col2=c2, ...
            n_trials=n_trials // 2,                           # :114
        )
```

- Chạy n_trials/2 lần với (col1, col2) và n_trials/2 lần với (col2, col1)
- Giảm position bias (LLM có thể prefer first/last document)
- Yêu cầu n_trials phải chẵn

---

## 6. Confidence via Logprobs

### Filter cascade confidence

**Location**: `lm.py:517-548`

```python
def format_logprobs_for_filter_cascade(self, logprobs):
    def get_normalized_true_prob(token_probs):
        if "True" in token_probs and "False" in token_probs:  # :525
            true_prob = token_probs["True"]                    # :526
            false_prob = token_probs["False"]                  # :527
            return true_prob / (true_prob + false_prob)         # :528
```

- Extract True/False probabilities từ top logprobs
- Normalize: `P(True) / (P(True) + P(False))` (`lm.py:528`)
- Dùng cho cascade routing decisions

### TopK cascade confidence

`compare_batch_binary_cascade` (`sem_topk.py:176-273`):
```python
for idx_j in range(len(helper_tokens[idx]) - 1, -1, -1):
    if helper_tokens[idx][idx_j].strip(" \n").isnumeric():    # :249
        conf = helper_confidences[idx][idx_j]                  # :250
        if conf >= cascade_threshold:                          # :251
            high_conf_idxs.add(idx)                            # :252
```

Tìm token numeric cuối cùng (document number) và check confidence.

---

## 7. Pairwise Evaluation trong sem_topk

Các sorting algorithms dùng pairwise LLM comparison:

- **quicksort** (`sem_topk.py:347-488`): Partition-based, partial sort cho top-K
- **heapsort** (`sem_topk.py:560-621`): `HeapDoc.__lt__` gọi LLM (`sem_topk.py:526-557`)
- **naive sort** (`sem_topk.py:276-344`): O(N^2) all-pairs voting

---

## 8. Kết luận

LLM as judge là pattern xuyên suốt LOTUS:
- Boolean judge (filter, join)
- Pairwise comparator (topk)
- Explicit evaluation (llm_as_judge, pairwise_judge)
- Confidence estimation via logprobs
- Position bias mitigation via permute_cols
