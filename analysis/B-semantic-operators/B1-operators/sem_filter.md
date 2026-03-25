# SEM_FILTER — `sem_filter`

## Metadata
- **File**: `lotus/sem_ops/sem_filter.py`
- **Accessor line**: 225 (`@pd.api.extensions.register_dataframe_accessor("sem_filter")`)
- **Core function line**: 24 (`def sem_filter(...)`)
- **Type**: Unary operator
- **Requires**: `lotus.settings.lm` (sem_filter.py:351)

## 1. Purpose & Use Cases

Loc cac row trong DataFrame dua tren dieu kien ngon ngu tu nhien. Moi row duoc danh gia True/False boi LLM.

**Use cases:**
- Loc sentiment: `df.sem_filter("The review {text} reflects a positive sentiment")`
- Loc theo dieu kien phuc tap: `df.sem_filter("The {description} mentions a safety concern")`
- Loc voi reasoning: dung `strategy=ReasoningStrategy.ZS_COT` de co giai thich

## 2. Call Stack Trace

```
1. SemFilterDataframe.__call__()                    # sem_filter.py:334
2.   lotus.nl_expression.parse_cols(user_instruction)  # sem_filter.py:358
3.   task_instructions.df2multimodal_info(df, col_li)  # sem_filter.py:367
4.   lotus.nl_expression.nle2str(user_instruction)     # sem_filter.py:369
5.   [Nếu có examples] df2multimodal_info(examples)    # sem_filter.py:376
6.   [Nếu cascade_args != None]:
6a.    ProxyModel.HELPER_LM → sem_filter() với helper_lm  # sem_filter.py:415-428
6b.    ProxyModel.EMBEDDING_MODEL → sem_search()           # sem_filter.py:440
6c.    importance_sampling(proxy_scores, cascade_args)      # sem_filter.py:443
6d.    learn_filter_cascade_thresholds()                    # sem_filter.py:449-462
6e.    Phân chia high_conf / low_conf                       # sem_filter.py:471-484
6f.    sem_filter() cho low_conf samples                    # sem_filter.py:510-522
7.   [Nếu không cascade]:
7a.    sem_filter(multimodal_data, lm, ...)                 # sem_filter.py:533-546
8.   Core sem_filter():
8a.    filter_formatter() cho mỗi doc                       # sem_filter.py:93-104
8b.    model(inputs, ...)                                   # sem_filter.py:112-114
8c.    filter_postprocess(lm_output.outputs)                # sem_filter.py:116
9.   Return: filtered DataFrame or (DataFrame, stats)       # sem_filter.py:590-593
```

## 3. Prompt Template (COPY VERBATIM)

### System instruction (task_instructions.py:99-101):
```
The user will provide a claim and some relevant context.
    Your job is to determine whether the claim is true for the given context.
```

### Non-CoT format (task_instructions.py:33-37):
```
Use the following format to provide your answer:
            Answer: <Your answer here. The answer should be either True or False>
```

### CoT format (task_instructions.py:25-30):
```
Let's think step by step. Use the following format to provide your answer:
        Reasoning:
<Your reasoning here. {reasoning_instructions}>

Answer: <Your answer here. The answer should be either True or False>
```

### User message format (task_instructions.py:68-84):
```
Context:
{serialized row data}

Claim: {user_instruction}
```

## 4. LLM Interaction

- **Model**: `lotus.settings.lm` (sem_filter.py:351)
- **Call**: `model(inputs, show_progress_bar=..., progress_bar_desc=..., **kwargs)` (sem_filter.py:112-114)
- **Logprobs**: ho tro qua `logprobs=True` parameter (sem_filter.py:105)
- **Postprocessing**: `filter_postprocess()` (postprocessors.py:182-218)
  - Tim "True" hoac "False" trong output (postprocessors.py:205-210)
  - Default value neu parse that bai (postprocessors.py:202-211)
  - Ho tro CoT postprocessor cho DeepSeek (postprocessors.py:46-93)

## 5. Optimization

| Feature | Status | Chi tiet |
|---|---|---|
| Batching | Yes | Toan bo inputs gui 1 lan qua `model(inputs)` (sem_filter.py:112-114) |
| Caching | Yes | `@operator_cache` decorator (sem_filter.py:333) |
| Cascading | Yes | 2 proxy models: HELPER_LM (sem_filter.py:407) va EMBEDDING_MODEL (sem_filter.py:435) |
| Early-termination | No | |
| Sampling | Yes | `importance_sampling()` cho cascade threshold learning (sem_filter.py:443) |
| Safe mode | Yes | Uoc tinh cost truoc khi chay (sem_filter.py:107-110) |

### Chi tiet Cascade:
1. **HELPER_LM** (sem_filter.py:407-434): Dung `lotus.settings.helper_lm` chay truoc, lay logprobs. Calibrate qua `calibrate_llm_logprobs()` (cascade_utils.py:33-39).
2. **EMBEDDING_MODEL** (sem_filter.py:435-441): Dung `sem_search` de lay similarity scores lam proxy.
3. **Threshold learning** (sem_filter.py:132-222): `learn_filter_cascade_thresholds()` chay oracle LLM tren sample, dung `learn_cascade_thresholds()` (cascade_utils.py:42-144) tim optimal (pos_threshold, neg_threshold).
4. **Routing**: High confidence → dung proxy result (sem_filter.py:475-485). Low confidence → gui den oracle LLM (sem_filter.py:507-527).

## 6. Input/Output Contract

### Input:
- `user_instruction: str` — Langex expression voi `{column}` placeholders
- `return_all: bool` — Neu True, tra ve tat ca rows voi column `filter_label` (sem_filter.py:565-578)
- `default: bool` — Gia tri mac dinh khi parse that bai (sem_filter.py:340, default=True)
- `examples: pd.DataFrame` — Phai co column "Answer" (sem_filter.py:375)
- `cascade_args: CascadeArgs` — Yeu cau `recall_target`, `precision_target`, `failure_probability` (sem_filter.py:398-404)

### Output:
- **return_all=False** (default): DataFrame chi giu rows co output=True (sem_filter.py:551-564)
- **return_all=True**: DataFrame goc + column `filter_label` (bool) (sem_filter.py:576-578)
- **return_explanations=True**: Them column `explanation_filter` (sem_filter.py:583-586)
- **return_raw_outputs=True**: Them column `raw_output_filter` (sem_filter.py:584-588)
- **return_stats=True**: Tra ve tuple (DataFrame, stats_dict) (sem_filter.py:590-591)

## 7. Edge Cases

1. **Column khong ton tai**: Raise `ValueError` (sem_filter.py:363-365)
2. **LM chua configure**: Raise `ValueError` (sem_filter.py:351-354)
3. **Parse that bai**: Dung `default` value (postprocessors.py:202-211)
4. **Cascade voi CoT**: Raise `ValueError` — CoT khong ho tro cho helper models (sem_filter.py:411-412)
5. **Helper LM chua set**: Raise `ValueError` khi dung ProxyModel.HELPER_LM (sem_filter.py:408-409)
6. **RM chua set**: Raise `ValueError` khi dung ProxyModel.EMBEDDING_MODEL (sem_filter.py:436-437)
7. **Column trung ten**: `get_out_col_name()` them suffix `_1`, `_2`... (sem_filter.py:567-574)

## 8. Code Examples

```python
# Basic filter
df.sem_filter("The {text} has positive sentiment")

# Voi CoT reasoning
df.sem_filter(
    "The {text} mentions safety issues",
    strategy=ReasoningStrategy.ZS_COT,
    return_explanations=True,
    return_all=True
)

# Voi cascade
from lotus.types import CascadeArgs, ProxyModel
cascade = CascadeArgs(
    recall_target=0.9,
    precision_target=0.9,
    failure_probability=0.2,
    sampling_percentage=0.1,
    proxy_model=ProxyModel.HELPER_LM
)
df.sem_filter("The {text} is relevant", cascade_args=cascade)

# Voi few-shot examples
examples = pd.DataFrame({
    "text": ["Great!", "Terrible"],
    "Answer": [True, False]
})
df.sem_filter("The {text} is positive", examples=examples)
```

## 9. Assessment

### Diem manh:
- **Cascade system** rat tinh vi: ho tro 2 proxy models, importance sampling, statistical threshold learning voi recall/precision targets
- **Flexibility**: Ho tro few-shot, CoT, ZS-CoT, custom default values
- **Production-ready**: Safe mode, progress bars, stats tracking

### Diem yeu:
- **Cascade complexity**: Code cascade chiem >60% accessor (sem_filter.py:382-530), kho maintain
- **Default=True**: Mac dinh la True khi parse that bai — co the dan den false positives
- **Single column limitation**: Cascade voi EMBEDDING_MODEL chi dung `col_li[0]` (sem_filter.py:440) — TODO comment ghi nhan van de nay
- **CoT + Cascade**: Khong tuong thich, bi raise error truc tiep (sem_filter.py:411-412)
