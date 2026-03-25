# E5 - Calibration and Confidence

## Tổng quan

LOTUS sử dụng logprobs cho cascade routing decisions. Confidence estimation qua True/False probability normalization và quantile-based calibration. Confidence information **không** exposed cho end user.

---

## 1. Logprobs cho Filter Cascade

**Location**: `lm.py:517-548`

```python
def format_logprobs_for_filter_cascade(self, logprobs):
```

### True/False Probability Extraction

```python
def get_normalized_true_prob(token_probs):
    if "True" in token_probs and "False" in token_probs:  # :525
        true_prob = token_probs["True"]                    # :526
        false_prob = token_probs["False"]                  # :527
        return true_prob / (true_prob + false_prob)         # :528
    return None                                             # :529
```

- Extract probabilities từ top_logprobs (`top_logprobs=10` mặc định, `lm.py:134`)
- Tìm "True" và "False" tokens trong logprobs
- Normalize: `P(True) / (P(True) + P(False))` → [0, 1]
- Nếu không tìm thấy cả True lẫn False: default dựa trên output text (`lm.py:541-542`)

### Scanning all tokens

```python
for resp_idx, response_logprobs in enumerate(logprobs):
    true_prob = None
    for logprob in response_logprobs:                               # :534
        token_probs = {
            top.token: np.exp(top.logprob)                          # :535
            for top in logprob.top_logprobs
        }
        true_prob = get_normalized_true_prob(token_probs)           # :536
        if true_prob is not None:
            break                                                    # :538
```

Scan qua tất cả tokens cho đến khi tìm được token position có cả True và False trong top_logprobs.

---

## 2. calibrate_llm_logprobs

**Location**: `cascade_utils.py:33-39`

```python
def calibrate_llm_logprobs(true_probs, cascade_args):
    num_quantiles = cascade_args.cascade_num_calibration_quantiles      # :35
    quantile_values = np.percentile(
        true_probs, np.linspace(0, 100, num_quantiles + 1)             # :36
    )
    true_probs = list(
        (np.digitize(true_probs, quantile_values) - 1) / num_quantiles  # :37
    )
    true_probs = list(np.clip(true_probs, 0, 1))                       # :38
```

### Quantile-based Calibration

Mục đích: transform raw logprobs thành calibrated scores.

Process:
1. Chia distribution thành `num_quantiles` (default 50) quantiles
2. Map mỗi probability thành quantile rank / num_quantiles
3. Clip to [0, 1]

Ví dụ: Nếu true_prob = 0.95 nằm ở quantile thứ 48/50 → calibrated = 48/50 = 0.96

Tại sao cần calibration:
- LLM logprobs thường **overconfident** (hầu hết gần 0 hoặc 1)
- Quantile calibration spreads scores ra → better threshold selection
- Default 50 quantiles cho granularity tốt

---

## 3. Cascade Logprobs cho TopK

**Location**: `lm.py:507-515`

```python
def format_logprobs_for_cascade(self, logprobs):
    all_tokens = []
    all_confidences = []
    for resp_logprobs in logprobs:
        tokens = [logprob.token for logprob in resp_logprobs]            # :511
        confidences = [np.exp(logprob.logprob) for logprob in resp_logprobs]  # :512
        all_tokens.append(tokens)
        all_confidences.append(confidences)
    return LogprobsForCascade(tokens=all_tokens, confidences=all_confidences)
```

Dùng cho sem_topk cascade: confidence of document number token.

---

## 4. Usage: Cascade Routing

### Filter cascade (`sem_filter.py:467-485`)
```python
for idx_i in range(len(proxy_scores)):
    true_prob = proxy_scores[idx_i]
    if true_prob >= pos_cascade_threshold:        # :477 - high confidence True
        proxy_outputs[idx_i] = True
    elif true_prob <= neg_cascade_threshold:       # :477 - high confidence False
        proxy_outputs[idx_i] = False
    # else: send to large LM
```

### TopK cascade (`sem_topk.py:248-252`)
```python
if helper_tokens[idx][idx_j].strip(" \n").isnumeric():
    conf = helper_confidences[idx][idx_j]
    if conf >= cascade_threshold:
        high_conf_idxs.add(idx)  # Accept helper result
```

---

## 5. Không Exposed cho User

Confidence information chỉ dùng internally:
- Không có `return_confidence` parameter cho sem_filter
- Không có confidence column trong output DataFrame
- User không thể filter by confidence
- Không có API để access raw logprobs

---

## 6. Kết luận

Calibration/Confidence trong LOTUS:
- **Logprobs**: True/False probability normalization (`lm.py:524-529`)
- **Calibration**: Quantile-based transformation (`cascade_utils.py:33-39`)
- **Usage**: Internal cascade routing only
- **Not exposed**: User cannot access confidence scores
- **Gap**: No user-facing confidence mechanism
