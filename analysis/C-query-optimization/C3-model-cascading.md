# C3 - Model Cascading trong LOTUS — DEEP DIVE

## Khai niem chung

Model cascading la ky thuat **dung proxy model re truoc, chi gui cac truong hop khong chac chan den oracle model dat**.
Day la optimization cot loi cua LOTUS, giup giam chi phi LLM 50-90% ma van dam bao do chinh xac thong ke.

---

## 1. Cau hinh Cascade

### CascadeArgs — `types.py:155-176`

```python
class CascadeArgs(BaseModel):
    recall_target: float = 0.8        # Muc tieu recall toi thieu
    precision_target: float = 0.8     # Muc tieu precision toi thieu
    sampling_percentage: float = 0.1  # Ty le sample de hoc thresholds
    failure_probability: float = 0.2  # Xac suat that bai cua dam bao thong ke
    map_instruction: str | None = None      # Instruction cho Map trong MSF
    map_examples: pd.DataFrame | None = None
    proxy_model: ProxyModel = ProxyModel.HELPER_LM  # Loai proxy

    # Filter cascade
    cascade_IS_weight: float = 0.9              # Trong so importance sampling
    cascade_num_calibration_quantiles: int = 50 # So quantiles cho calibration

    # Join cascade
    min_join_cascade_size: int = 100            # Toi thieu de kich hoat cascade
    cascade_IS_max_sample_range: int = 200      # Gioi han sample range
    cascade_IS_random_seed: int | None = None   # Seed cho reproducibility
```

### ProxyModel — `types.py:150-152`

```python
class ProxyModel(Enum):
    HELPER_LM = "helper_lm"          # Dung LLM nho (vd: gpt-4o-mini) voi logprobs
    EMBEDDING_MODEL = "embedding_model"  # Dung embedding similarity scores
```

---

## 2. sem_filter Cascade — `sem_filter.py:383-530`

### Luong xu ly chi tiet

#### Buoc 1: Lay proxy scores

**Truong hop HELPER_LM** (`sem_filter.py:407-434`):
1. Kiem tra `lotus.settings.helper_lm` co ton tai — `sem_filter.py:408-409`
2. Chay `sem_filter()` voi helper_lm va `logprobs=True` — `sem_filter.py:415-428`
3. Extract logprobs: `format_logprobs_for_filter_cascade` — `lm.py:517-548`
   - Tim top_logprobs chua "True" va "False" — `lm.py:525-528`
   - Tinh `true_prob = P(True) / (P(True) + P(False))` — `lm.py:528`
   - Default: 1 neu "True" trong tokens, 0 neu khong — `lm.py:541-542`
4. Calibrate logprobs: `calibrate_llm_logprobs` — `cascade_utils.py:33-39`

**Truong hop EMBEDDING_MODEL** (`sem_filter.py:435-441`):
1. Kiem tra `lotus.settings.rm` ton tai — `sem_filter.py:436-437`
2. Chay `sem_search` voi K=len(df) de lay similarity scores cho tat ca rows — `sem_filter.py:440`
3. Su dung `vec_scores_sim_score` lam proxy scores — `sem_filter.py:441`

#### Buoc 2: Calibrate LLM logprobs — `cascade_utils.py:33-39`

```python
def calibrate_llm_logprobs(true_probs, cascade_args):
    num_quantiles = cascade_args.cascade_num_calibration_quantiles  # default 50
    quantile_values = np.percentile(true_probs, np.linspace(0, 100, num_quantiles + 1))
    true_probs = list((np.digitize(true_probs, quantile_values) - 1) / num_quantiles)
    true_probs = list(np.clip(true_probs, 0, 1))
    return true_probs
```

**Muc dich:** LLM logprobs thuong khong duoc calibrated tot (overconfident hoac underconfident).
Quantile calibration chuyen doi ve uniform distribution tren [0, 1], giup threshold learning hoat dong tot hon.

#### Buoc 3: Importance Sampling — `cascade_utils.py:8-30`

```python
def importance_sampling(proxy_scores, cascade_args):
    w = np.sqrt(proxy_scores)
    is_weight = cascade_args.cascade_IS_weight  # default 0.9
    w = is_weight * w / np.sum(w) + (1 - is_weight) * np.ones(len(proxy_scores)) / len(proxy_scores)

    sample_range = min(cascade_args.cascade_IS_max_sample_range, len(proxy_scores))
    sample_w = w[:sample_range]
    sample_w = sample_w / np.sum(sample_w)

    sample_size = int(cascade_args.sampling_percentage * len(proxy_scores))
    sample_indices = np.random.choice(indices, sample_size, p=sample_w)

    correction_factors = (1 / len(proxy_scores)) / w
    return sample_indices, correction_factors
```

**Muc dich:** Chon sample KHONG DONG DEU — uu tien cac items co proxy score cao (nhieu khả nang la positive).
`correction_factors` dung de dieu chinh lai bias trong uoc luong recall/precision.

**Chi tiet:**
- `w = sqrt(proxy_scores)` — trong so ty le voi can bac hai cua score
- Mixture: 90% weighted + 10% uniform — `cascade_utils.py:18`
- `sample_range` gioi han chi lay `cascade_IS_max_sample_range` (200) items dau tien — `cascade_utils.py:20`
- `correction_factors = uniform_weight / actual_weight` — `cascade_utils.py:28`

#### Buoc 4: Hoc Cascade Thresholds — `cascade_utils.py:42-144`

**Input:**
- `proxy_scores`: scores tu proxy model cho sample
- `oracle_outputs`: True/False tu oracle LM cho sample
- `sample_correction_factors`: tu importance sampling
- `cascade_args`: targets

**Quy trinh:**

1. **Pair data va sort** — `cascade_utils.py:96-97`:
   ```python
   paired_data = list(zip(proxy_scores, oracle_outputs, sample_correction_factors))
   sorted_pairs = sorted(paired_data, key=lambda x: x[0], reverse=True)
   ```

2. **Khoi tao** `best_combination = (1.0, 0.0)` — `cascade_utils.py:100`
   - tau_pos = 1.0 (accept nothing), tau_neg = 0.0 (reject nothing)

3. **Tim tau_neg dua tren recall target** — `cascade_utils.py:103-104`:
   ```python
   tau_neg_0 = calculate_tau_neg(sorted_pairs, 1.0, cascade_args.recall_target)
   ```
   - `calculate_tau_neg` tim tau_neg nho nhat sao cho `recall(tau_pos, tau_neg) >= recall_target` — `cascade_utils.py:88-93`

4. **Statistical correction cho recall** — `cascade_utils.py:107-124`:
   - Chia data thanh 2 nhom: Z1 (scores >= tau_neg) va Z2 (scores < tau_neg)
   - Tinh `UB(Z1)` va `LB(Z2)` dung Hoeffding-style bounds — `cascade_utils.py:115-116`
   - `corrected_recall_target = UB(Z1) / (UB(Z1) + LB(Z2))` — `cascade_utils.py:120`
   - Tim lai tau_neg voi corrected target — `cascade_utils.py:123-124`

5. **Statistical correction cho precision** — `cascade_utils.py:127-137`:
   - Duyet tat ca candidate thresholds
   - Cho moi threshold, tinh `LB(precision)` voi confidence `failure_probability / len(sorted_pairs)`
   - Chon threshold nho nhat ma `LB > precision_target` — `cascade_utils.py:134-135`
   - `tau_pos = max(tau_neg, min(candidate_thresholds))` — `cascade_utils.py:137`

#### Buoc 5: Ap dung Thresholds — `sem_filter.py:467-530`

```python
# sem_filter.py:475-485
for idx_i in range(len(proxy_scores)):
    true_prob = proxy_scores[idx_i]
    if true_prob >= pos_cascade_threshold or true_prob <= neg_cascade_threshold:
        high_conf_idxs.add(idx_i)
        proxy_outputs[idx_i] = True if true_prob >= pos_cascade_threshold else False
```

- **High confidence positive** (score >= tau_pos): accept, danh dau True
- **High confidence negative** (score <= tau_neg): reject, danh dau False
- **Low confidence** (tau_neg < score < tau_pos): gui den oracle LM — `sem_filter.py:507-522`

---

## 3. sem_join Cascade — `sem_join.py:180-333`

### Luong xu ly

#### Buoc 1: join_optimizer — `sem_join.py:242-257`
Goi `join_optimizer` de chon strategy tot nhat (xem C2 chi tiet).

#### Buoc 2: run_sem_sim_join lam helper — `sem_join.py:336-366`
```python
def run_sem_sim_join(l1, l2, col1_label, col2_label):
    l2_df = l2.to_frame(name=col2_label)
    l2_df = l2_df.sem_index(col2_label, f"{col2_label}_index")
    K = len(l2)
    out = l1_df.sem_sim_join(l2_df, left_on=col1_label, right_on=col2_label, K=K)
    out["_scores"] = calibrate_sem_sim_join(out["_scores"].tolist())
    return out
```
- Index l2 (embedding), roi tim K nearest neighbors cho moi item trong l1
- `calibrate_sem_sim_join` — `cascade_utils.py:147-149` — clip scores ve [0, 1]

#### Buoc 3: learn_join_cascade_threshold — `sem_join.py:530-603`
- Sample tu helper join results — `sem_join.py:564-571`
- Chay oracle LM tren sample — `sem_join.py:577-587`
- Tim optimal thresholds qua `learn_cascade_thresholds` — `sem_join.py:589-594`
- Fallback: neu loi, tra ve (1.0, 0.0) — full join — `sem_join.py:598-601`

#### Buoc 4: Route to oracle — `sem_join.py:266-312`
- High confidence → accept truc tiep — `sem_join.py:267`
- Low confidence → chay `sem_filter` voi oracle LM — `sem_join.py:291-301`

### Stats tracking — `sem_join.py:318-325`
```python
stats = {
    "join_resolved_by_helper_model": num_helper + num_helper_high_conf_neg,
    "join_helper_positive": num_helper,
    "join_helper_negative": num_helper_high_conf_neg,
    "join_resolved_by_large_model": num_large,
    "optimized_join_cost": join_optimization_cost,
    "total_LM_calls": join_optimization_cost + num_large,
}
```

---

## 4. sem_topk Cascade — `sem_topk.py:176-273`

### Khac biet voi filter/join cascade

TopK cascade **khong hoc thresholds truoc**. Thay vao do, `cascade_threshold` la tham so user cung cap truc tiep.

### compare_batch_binary_cascade — `sem_topk.py:176-273`

```python
def compare_batch_binary_cascade(pairs, model, user_instruction, cascade_threshold, strategy=None):
```

**Luong xu ly:**

1. **Tao prompts** cho tat ca pairs — `sem_topk.py:219-223`

2. **Chay helper_lm voi logprobs** — `sem_topk.py:231`:
   ```python
   helper_output = helper_lm(match_prompts, kwargs={"logprobs": True})
   ```

3. **Format logprobs** — `sem_topk.py:235`:
   ```python
   formatted_logprobs = helper_lm.format_logprobs_for_cascade(helper_logprobs)
   ```
   - `format_logprobs_for_cascade` tai `lm.py:507-515` — extract tokens va confidence values

4. **Check confidence** — `sem_topk.py:247-252`:
   ```python
   for idx_j in range(len(helper_tokens[idx]) - 1, -1, -1):
       if helper_tokens[idx][idx_j].strip(" \n").isnumeric():
           conf = helper_confidences[idx][idx_j]
           if conf >= cascade_threshold:
               high_conf_idxs.add(idx)
   ```
   - Tim token so (1 hoac 2) tu cuoi len
   - Neu confidence cua token do >= threshold → accept helper result

5. **Gui low-confidence den large LM** — `sem_topk.py:257-272`:
   ```python
   if len(high_conf_idxs) != len(helper_logprobs):
       low_conf_idxs = sorted([i for i in range(len(helper_logprobs)) if i not in high_conf_idxs])
       large_lm_results = model(large_match_prompts)
       for idx, res in enumerate(large_lm_results.outputs):
           parsed_results[low_conf_idxs[idx]] = parse_ans_binary(res)
   ```

### Su dung trong quicksort — `sem_topk.py:437-455`

Cascade duoc ap dung trong moi partition step cua quicksort:
```python
# sem_topk.py:438-444
comparisons, explanations, small_tokens, large_tokens, num_large_calls = compare_batch_binary_cascade(
    pairs, model, user_instruction, cascade_threshold, strategy=strategy
)
```

---

## 5. Co so Toan hoc

### UB/LB Bounds — `cascade_utils.py:52-56`

```python
def UB(mean, std_dev, s, delta):
    return mean + (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5)

def LB(mean, std_dev, s, delta):
    return mean - (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5)
```

Day la **Hoeffding-style concentration inequalities**:
- Voi xac suat >= 1 - delta:
  - Gia tri thuc <= UB(sample_mean, sample_std, sample_size, delta)
  - Gia tri thuc >= LB(sample_mean, sample_std, sample_size, delta)
- `s` = sample_size, `delta` = failure_probability / 2

### Corrected Recall Target — `cascade_utils.py:118-121`

```python
corrected_recall_target = ub_z1 / (ub_z1 + lb_z2)
```

- Z1 = weighted correct trong nhom scores >= tau_neg (recall numerator)
- Z2 = weighted correct trong nhom scores < tau_neg (missed items)
- UB(Z1) la gioi han tren cua recall numerator
- LB(Z2) la gioi han duoi cua missed items
- Corrected target dam bao: voi prob >= 1 - delta, recall thuc te >= recall_target

### Precision Guarantee — `cascade_utils.py:127-137`

```python
for pair in sorted_pairs:
    possible_threshold = pair[0]
    Z = [int(x[1]) for x in sorted_pairs if x[0] >= possible_threshold]
    mean_z = float(np.mean(Z))
    std_z = float(np.std(Z))
    p_l = LB(mean_z, std_z, len(Z), cascade_args.failure_probability / len(sorted_pairs))
    if p_l > cascade_args.precision_target:
        candidate_thresholds.append(possible_threshold)
```

- Cho moi candidate tau_pos, tinh LB cua precision cua nhom accepted
- Dung Bonferroni correction: delta / len(sorted_pairs) cho moi test
- Chon threshold nho nhat (accept nhieu nhat) ma van dam bao precision

---

## 6. So sanh cac Cascade approaches

| Khia canh | sem_filter | sem_join | sem_topk |
|-----------|-----------|----------|----------|
| Proxy model | Helper LM logprobs HOAC embedding similarity | Embedding similarity (sem_sim_join) | Helper LM logprobs |
| Threshold learning | Co, tu sample | Co, tu sample | Khong, user cung cap |
| Statistical guarantees | Co (recall + precision) | Co (recall + precision) | Khong |
| Khi nao kich hoat | `cascade_args != None` | `cascade_args != None` va `num_pairs >= min_join_cascade_size` | `cascade_threshold != None` |
| File chinh | `sem_filter.py:383-530` | `sem_join.py:180-333` | `sem_topk.py:176-273` |
| Complexity | O(n) proxy + O(sample) oracle learning + O(low_conf) oracle | O(n*m) embedding + O(sample) oracle learning + O(low_conf) oracle | O(n*logn) helper calls + O(low_conf_per_partition) oracle |
