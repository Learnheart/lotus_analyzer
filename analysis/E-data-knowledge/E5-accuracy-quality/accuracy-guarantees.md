# E5 - Accuracy Guarantees

## Tổng quan

LOTUS cung cấp statistical accuracy guarantees thông qua CascadeArgs framework. Hệ thống học threshold tự động với Hoeffding-style bounds để đảm bảo recall/precision targets.

---

## 1. CascadeArgs Configuration

**Location**: `types.py:155-176`

```python
class CascadeArgs(BaseModel):
    recall_target: float = 0.8                      # :156
    precision_target: float = 0.8                    # :157
    sampling_percentage: float = 0.1                 # :158
    failure_probability: float = 0.2                 # :159
    map_instruction: str | None = None               # :160
    map_examples: pd.DataFrame | None = None         # :161
    proxy_model: ProxyModel = ProxyModel.HELPER_LM   # :162

    # Filter cascade args
    cascade_IS_weight: float = 0.9                   # :165
    cascade_num_calibration_quantiles: int = 50      # :166

    # Join cascade args
    min_join_cascade_size: int = 100                 # :169
    cascade_IS_max_sample_range: int = 200           # :170
    cascade_IS_random_seed: int | None = None        # :171
```

---

## 2. learn_cascade_thresholds

**Location**: `cascade_utils.py:42-144`

```python
def learn_cascade_thresholds(proxy_scores, oracle_outputs,
                              sample_correction_factors, cascade_args):
```

### Mathematical Basis: UB/LB Bounds

**Location**: `cascade_utils.py:52-56`

```python
def UB(mean, std_dev, s, delta):
    return float(mean + (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5))

def LB(mean, std_dev, s, delta):
    return float(mean - (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5))
```

Hoeffding-style concentration inequalities:
- `UB`: Upper bound = mean + confidence interval
- `LB`: Lower bound = mean - confidence interval
- `s`: sample size
- `delta`: failure probability

### Threshold Learning Algorithm

#### Step 1: Initial tau_neg from recall (`cascade_utils.py:103`)
```python
tau_neg_0 = calculate_tau_neg(sorted_pairs, best_combination[0], cascade_args.recall_target)
```

#### Step 2: Statistical correction for recall (`cascade_utils.py:107-124`)
```python
Z1 = [int(x[1]) * x[2] for x in sorted_pairs if x[0] >= best_combination[1]]
Z2 = [int(x[1]) * x[2] for x in sorted_pairs if x[0] < best_combination[1]]

ub_z1 = UB(mean_z1, std_z1, sample_size, cascade_args.failure_probability / 2)  # :115
lb_z2 = LB(mean_z2, std_z2, sample_size, cascade_args.failure_probability / 2)  # :116

corrected_recall_target = ub_z1 / (ub_z1 + lb_z2)                               # :120
tau_neg_prime = calculate_tau_neg(sorted_pairs, ..., corrected_recall_target)     # :123
```

Statistical correction tăng recall target để account for sampling uncertainty.

#### Step 3: Precision threshold (`cascade_utils.py:127-137`)
```python
for pair in sorted_pairs:
    possible_threshold = pair[0]
    Z = [int(x[1]) for x in sorted_pairs if x[0] >= possible_threshold]
    p_l = LB(mean_z, std_z, len(Z), cascade_args.failure_probability / len(sorted_pairs))  # :133
    if p_l > cascade_args.precision_target:                                                   # :134
        candidate_thresholds.append(possible_threshold)

best_combination = (max(best_combination[1], min(candidate_thresholds)), best_combination[1])  # :137
```

Tìm tau_pos thấp nhất mà precision lower bound vẫn vượt target.

---

## 3. Importance Sampling

**Location**: `cascade_utils.py:8-30`

```python
def importance_sampling(proxy_scores, cascade_args):
    w = np.sqrt(proxy_scores)                                              # :16
    is_weight = cascade_args.cascade_IS_weight                             # :17
    w = is_weight * w / np.sum(w) + (1 - is_weight) * np.ones(...) / N    # :18

    sample_size = int(cascade_args.sampling_percentage * len(proxy_scores))  # :25
    sample_indices = np.random.choice(indices, sample_size, p=sample_w)     # :26

    correction_factors = (1 / len(proxy_scores)) / w                       # :28
```

- Weighted sampling: bias toward higher proxy scores (`sqrt(proxy_scores)`)
- Mixing: `IS_weight * biased + (1 - IS_weight) * uniform` (`cascade_utils.py:18`)
- Correction factors: account for sampling bias in threshold learning
- Default `cascade_IS_weight=0.9` → 90% biased, 10% uniform

---

## 4. calibrate_llm_logprobs

**Location**: `cascade_utils.py:33-39`

```python
def calibrate_llm_logprobs(true_probs, cascade_args):
    num_quantiles = cascade_args.cascade_num_calibration_quantiles         # :35
    quantile_values = np.percentile(true_probs, np.linspace(0, 100, num_quantiles + 1))  # :36
    true_probs = list((np.digitize(true_probs, quantile_values) - 1) / num_quantiles)    # :37
    true_probs = list(np.clip(true_probs, 0, 1))                                         # :38
```

Quantile-based calibration:
1. Compute quantile boundaries (default 50 quantiles)
2. Map each probability to its quantile rank
3. Normalize to [0, 1]
4. Makes probabilities more uniformly distributed → better threshold learning

---

## 5. Guarantee Semantics

Với CascadeArgs(recall_target=0.9, precision_target=0.95, failure_probability=0.2):

- **Recall >= 0.9**: Probability(recall < 0.9) <= 0.2 (failure_probability)
- **Precision >= 0.95**: Probability(precision < 0.95) <= 0.2
- **Trade-off**: Higher targets → more LLM calls (fewer samples routed to helper)

---

## 6. Kết luận

LOTUS cung cấp principled accuracy guarantees:
- Statistical bounds (Hoeffding-style)
- Importance sampling cho efficient threshold learning
- Logprob calibration cho better proxy scores
- User-configurable recall/precision targets
- Failure probability control
