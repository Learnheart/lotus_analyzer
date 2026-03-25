# E5 - Evaluation and Benchmarks

## Tổng quan

LOTUS cung cấp evaluation tools (llm_as_judge, pairwise_judge) nhưng không include benchmark datasets trong source code. Testing infrastructure tồn tại nhưng focused on integration tests.

---

## 1. Evaluation Tools

### llm_as_judge

**Location**: `evals/llm_as_judge.py:16-114`

```python
def llm_as_judge(docs, model, judge_instruction,
                 response_format=None, n_trials=1,
                 system_prompt=None, ...):
```

Features:
- Custom judge instructions
- Configurable system prompt (default: "intelligent, rigorous, and fair evaluator") (`llm_as_judge.py:70-74`)
- n_trials for multiple evaluations (`llm_as_judge.py:22`)
- Response format support via Pydantic BaseModel (`llm_as_judge.py:21`)
- Cache disabled during evaluation (`llm_as_judge.py:82`)
- Few-shot examples support (`llm_as_judge.py:25-27`)

### pairwise_judge

**Location**: `evals/pairwise_judge.py:13-158`

```python
class PairwiseJudgeDataframe:
    def __call__(self, col1, col2, judge_instruction,
                 n_trials=1, permute_cols=False, ...):
```

Features:
- Compare two columns side-by-side
- permute_cols for position bias mitigation (`pairwise_judge.py:100-140`)
- Internally uses llm_as_judge (`pairwise_judge.py:142-158`)

---

## 2. Test Infrastructure

### Test files location
```
tests/
.github/tests/
```

Tests focused on:
- Integration testing (operators work correctly)
- API compatibility
- Edge cases

### Không có Benchmark Datasets

Không tìm thấy trong source code:
- Standard benchmark datasets (BEIR, MTEB, etc.)
- Evaluation scripts cho accuracy metrics
- Comparison against baselines
- Performance benchmarking code

---

## 3. Custom Evaluation Pattern

User có thể tạo evaluation pipeline:

```python
# Evaluate sem_filter accuracy
gold_labels = pd.DataFrame({
    "text": ["positive review", "negative review"],
    "gold": [True, False]
})

results = gold_labels.sem_filter("the {text} has positive sentiment", return_all=True)
accuracy = (results["filter_label"] == gold_labels["gold"]).mean()
```

Hoặc dùng llm_as_judge:
```python
results = df.llm_as_judge(
    "Rate the quality of {output} on a scale of 1-5",
    n_trials=3
)
```

---

## 4. Metrics Available

LOTUS không compute metrics internally, nhưng cascade stats provide proxy metrics:

- `filters_resolved_by_helper_model` / `filters_resolved_by_large_model` (`sem_filter.py:468-469`)
- `join_resolved_by_helper_model` / `join_resolved_by_large_model` (`sem_join.py:318-319`)
- `total_LM_calls` (`sem_join.py:324`)
- Token usage: `model.stats.virtual_usage` / `model.stats.physical_usage` (`lm.py:477-483`)

---

## 5. Kết luận

Evaluation trong LOTUS:
- **Tools**: llm_as_judge, pairwise_judge cho custom evaluation
- **No benchmarks**: Không có standard benchmark datasets
- **Test focus**: Integration testing, not accuracy benchmarking
- **Custom eval**: User phải tự build evaluation pipelines
- **Stats available**: Cascade routing stats, token usage
