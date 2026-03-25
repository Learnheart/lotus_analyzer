# E6 - Cost-Accuracy Tradeoff

## Tổng quan

LOTUS cung cấp cascade mechanism cho cost-accuracy tradeoff: helper_lm (small, cheap) xử lý high-confidence cases, lm (large, expensive) xử lý low-confidence cases. User control tradeoff qua CascadeArgs.

---

## 1. Cascade Architecture

### Two-model Setup

```python
lotus.settings.configure(
    lm=LM(model="gpt-4o"),           # Large, expensive, accurate
    helper_lm=LM(model="gpt-4o-mini") # Small, cheap, less accurate
)
```

### Routing Logic (`sem_filter.py:467-530`)

```python
for idx_i in range(len(proxy_scores)):
    true_prob = proxy_scores[idx_i]
    if true_prob >= pos_cascade_threshold:     # :477 - confident True
        proxy_outputs[idx_i] = True
        high_conf_idxs.add(idx_i)
    elif true_prob <= neg_cascade_threshold:   # :477 - confident False
        proxy_outputs[idx_i] = False
        high_conf_idxs.add(idx_i)
    # else: low confidence → send to large LM

# Send low confidence to large LM
low_conf_multimodal_data = [multimodal_data[idx] for idx in low_conf_idxs]  # :508
large_output = sem_filter(low_conf_multimodal_data, lotus.settings.lm, ...)  # :510-522
```

---

## 2. CascadeArgs Knobs

**Location**: `types.py:155-171`

| Parameter | Default | Effect |
|---|---|---|
| `recall_target` | 0.8 | Higher → more items sent to large LM | `types.py:156` |
| `precision_target` | 0.8 | Higher → stricter positive threshold | `types.py:157` |
| `sampling_percentage` | 0.1 | Higher → more accurate thresholds, more cost | `types.py:158` |
| `failure_probability` | 0.2 | Lower → stronger guarantees, more conservative | `types.py:159` |

### Tradeoff dynamics

- **recall_target = 1.0**: Tất cả positives phải được giữ → conservative tau_neg → nhiều items đến large LM
- **precision_target = 1.0**: Tất cả accepted positives phải đúng → strict tau_pos → nhiều items đến large LM
- **Cả hai = 1.0**: Gần như tất cả đến large LM → no cost savings, maximum accuracy
- **Cả hai = 0.5**: Aggressive cascade → significant cost savings, lower accuracy

---

## 3. sem_topk cascade_threshold

**Location**: `sem_topk.py:176-273`

```python
def compare_batch_binary_cascade(pairs, model, user_instruction,
                                  cascade_threshold, ...):
```

Cho sem_topk, cascade_threshold là **single value** (không phải pair):
- Helper LM makes comparisons
- Check confidence at document number token
- Low confidence → send to large LM

```python
# sem_topk.py:248-252
for idx_j in range(len(helper_tokens[idx]) - 1, -1, -1):
    if helper_tokens[idx][idx_j].strip(" \n").isnumeric():
        conf = helper_confidences[idx][idx_j]
        if conf >= cascade_threshold:
            high_conf_idxs.add(idx)
```

---

## 4. Default: No Cascade

Khi `cascade_args=None` (default), không có cascade (`sem_filter.py:532-549`):
```python
else:
    output = sem_filter(
        multimodal_data, lotus.settings.lm, ...  # Full accuracy, all to large LM
    )
```

**Full accuracy, higher cost** - mọi items gửi cho large LM.

---

## 5. Proxy Models

**Location**: `types.py:150-153`

```python
class ProxyModel(Enum):
    HELPER_LM = "helper_lm"           # :151
    EMBEDDING_MODEL = "embedding_model"  # :152
```

### HELPER_LM proxy (`sem_filter.py:407-434`)
```python
helper_output = sem_filter(
    multimodal_data, lotus.settings.helper_lm, ...,
    logprobs=True,                                          # :423
)
formatted_helper_logprobs = lotus.settings.helper_lm.format_logprobs_for_filter_cascade(...)  # :432
proxy_scores = calibrate_llm_logprobs(formatted_helper_logprobs.true_probs, cascade_args)     # :434
```

Cost: N helper LM calls + M large LM calls (M = low confidence count)

### EMBEDDING_MODEL proxy (`sem_filter.py:435-441`)
```python
search_df = self._obj.sem_search(col_li[0], formatted_usr_instr, K=len(self._obj), return_scores=True)  # :440
proxy_scores = search_df["vec_scores_sim_score"].tolist()                                                # :441
```

Cost: 1 embedding search + M large LM calls (cheaper proxy, potentially less accurate)

---

## 6. TPM/Rate Limiting cho Cost Control

### Rate limiting (`lm.py:74-75`)
```python
rate_limit: int | None = None,    # RPM limit
tpm_limit: int | None = None,     # TPM limit
```

Không giảm total cost, nhưng control throughput để stay within API tier limits.

---

## 7. Cost Comparison

| Scenario | Cost | Accuracy |
|---|---|---|
| No cascade | N * large_model_cost | Highest |
| Helper LM cascade | N * helper_cost + M * large_cost | Controlled via targets |
| Embedding cascade | search_cost + M * large_cost | Depends on embedding quality |
| All cascade knobs at 1.0 | ~= No cascade | ~= No cascade |
| Aggressive cascade (0.5) | Significantly lower | Lower, bounded by targets |

---

## 8. Kết luận

Cost-Accuracy tradeoff trong LOTUS:
- **Cascade mechanism**: helper_lm hoặc embedding model as proxy
- **User control**: recall_target, precision_target, sampling_percentage
- **Statistical guarantees**: Bounded failure probability
- **Default conservative**: No cascade (full accuracy)
- **Flexible**: Per-operator cascade configuration
- **TPM/rate limiting**: API tier compliance, not cost reduction
