# C3 - Model Cascading trong LOTUS — DEEP DIVE

## Khái niệm chung

Model cascading là kỹ thuật **dùng proxy model rẻ trước, chỉ gửi các trường hợp không chắc chắn đến oracle model đắt**.
Đây là optimization cốt lõi của LOTUS, giúp giảm chi phí LLM 50-90% mà vẫn đảm bảo độ chính xác thống kê.

---

## 1. Cấu hình Cascade

### CascadeArgs — `types.py:155-176`

```python
class CascadeArgs(BaseModel):
    recall_target: float = 0.8        # Mục tiêu recall tối thiểu
    precision_target: float = 0.8     # Mục tiêu precision tối thiểu
    sampling_percentage: float = 0.1  # Tỷ lệ sample để học thresholds
    failure_probability: float = 0.2  # Xác suất thất bại của đảm bảo thống kê
    map_instruction: str | None = None      # Instruction cho Map trong MSF
    map_examples: pd.DataFrame | None = None
    proxy_model: ProxyModel = ProxyModel.HELPER_LM  # Loại proxy

    # Filter cascade
    cascade_IS_weight: float = 0.9              # Trọng số importance sampling
    cascade_num_calibration_quantiles: int = 50 # Số quantiles cho calibration

    # Join cascade
    min_join_cascade_size: int = 100            # Tối thiểu để kích hoạt cascade
    cascade_IS_max_sample_range: int = 200      # Giới hạn sample range
    cascade_IS_random_seed: int | None = None   # Seed cho reproducibility
```

### ProxyModel — `types.py:150-152`

```python
class ProxyModel(Enum):
    HELPER_LM = "helper_lm"          # Dùng LLM nhỏ (vd: gpt-4o-mini) với logprobs
    EMBEDDING_MODEL = "embedding_model"  # Dùng embedding similarity scores
```

---

## 2. sem_filter Cascade — `sem_filter.py:383-530`

### Luồng xử lý chi tiết

#### Bước 1: Lấy proxy scores

**Trường hợp HELPER_LM** (`sem_filter.py:407-434`):
1. Kiểm tra `lotus.settings.helper_lm` có tồn tại — `sem_filter.py:408-409`
2. Chạy `sem_filter()` với helper_lm và `logprobs=True` — `sem_filter.py:415-428`
3. Extract logprobs: `format_logprobs_for_filter_cascade` — `lm.py:517-548`
   - Tìm top_logprobs chứa "True" và "False" — `lm.py:525-528`
   - Tính `true_prob = P(True) / (P(True) + P(False))` — `lm.py:528`
   - Default: 1 nếu "True" trong tokens, 0 nếu không — `lm.py:541-542`
4. Calibrate logprobs: `calibrate_llm_logprobs` — `cascade_utils.py:33-39`

**Trường hợp EMBEDDING_MODEL** (`sem_filter.py:435-441`):
1. Kiểm tra `lotus.settings.rm` tồn tại — `sem_filter.py:436-437`
2. Chạy `sem_search` với K=len(df) để lấy similarity scores cho tất cả rows — `sem_filter.py:440`
3. Sử dụng `vec_scores_sim_score` làm proxy scores — `sem_filter.py:441`

#### Bước 2: Calibrate LLM logprobs — `cascade_utils.py:33-39`

```python
def calibrate_llm_logprobs(true_probs, cascade_args):
    num_quantiles = cascade_args.cascade_num_calibration_quantiles  # default 50
    quantile_values = np.percentile(true_probs, np.linspace(0, 100, num_quantiles + 1))
    true_probs = list((np.digitize(true_probs, quantile_values) - 1) / num_quantiles)
    true_probs = list(np.clip(true_probs, 0, 1))
    return true_probs
```

**Mục đích:** LLM logprobs thường không được calibrated tốt (overconfident hoặc underconfident).
Quantile calibration chuyển đổi về uniform distribution trên [0, 1], giúp threshold learning hoạt động tốt hơn.

#### Bước 3: Importance Sampling — `cascade_utils.py:8-30`

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

**Mục đích:** Chọn sample KHÔNG ĐỒNG ĐỀU — ưu tiên các items có proxy score cao (nhiều khả năng là positive).
`correction_factors` dùng để điều chỉnh lại bias trong ước lượng recall/precision.

**Chi tiết:**
- `w = sqrt(proxy_scores)` — trọng số tỷ lệ với căn bậc hai của score
- Mixture: 90% weighted + 10% uniform — `cascade_utils.py:18`
- `sample_range` giới hạn chỉ lấy `cascade_IS_max_sample_range` (200) items đầu tiên — `cascade_utils.py:20`
- `correction_factors = uniform_weight / actual_weight` — `cascade_utils.py:28`

#### Bước 4: Học Cascade Thresholds — `cascade_utils.py:42-144`

**Input:**
- `proxy_scores`: scores từ proxy model cho sample
- `oracle_outputs`: True/False từ oracle LM cho sample
- `sample_correction_factors`: từ importance sampling
- `cascade_args`: targets

**Quy trình:**

1. **Pair data và sort** — `cascade_utils.py:96-97`:
   ```python
   paired_data = list(zip(proxy_scores, oracle_outputs, sample_correction_factors))
   sorted_pairs = sorted(paired_data, key=lambda x: x[0], reverse=True)
   ```

2. **Khởi tạo** `best_combination = (1.0, 0.0)` — `cascade_utils.py:100`
   - tau_pos = 1.0 (accept nothing), tau_neg = 0.0 (reject nothing)

3. **Tìm tau_neg dựa trên recall target** — `cascade_utils.py:103-104`:
   ```python
   tau_neg_0 = calculate_tau_neg(sorted_pairs, 1.0, cascade_args.recall_target)
   ```
   - `calculate_tau_neg` tìm tau_neg nhỏ nhất sao cho `recall(tau_pos, tau_neg) >= recall_target` — `cascade_utils.py:88-93`

4. **Statistical correction cho recall** — `cascade_utils.py:107-124`:
   - Chia data thành 2 nhóm: Z1 (scores >= tau_neg) và Z2 (scores < tau_neg)
   - Tính `UB(Z1)` và `LB(Z2)` dùng Hoeffding-style bounds — `cascade_utils.py:115-116`
   - `corrected_recall_target = UB(Z1) / (UB(Z1) + LB(Z2))` — `cascade_utils.py:120`
   - Tìm lại tau_neg với corrected target — `cascade_utils.py:123-124`

5. **Statistical correction cho precision** — `cascade_utils.py:127-137`:
   - Duyệt tất cả candidate thresholds
   - Cho mỗi threshold, tính `LB(precision)` với confidence `failure_probability / len(sorted_pairs)`
   - Chọn threshold nhỏ nhất mà `LB > precision_target` — `cascade_utils.py:134-135`
   - `tau_pos = max(tau_neg, min(candidate_thresholds))` — `cascade_utils.py:137`

#### Bước 5: Áp dụng Thresholds — `sem_filter.py:467-530`

```python
# sem_filter.py:475-485
for idx_i in range(len(proxy_scores)):
    true_prob = proxy_scores[idx_i]
    if true_prob >= pos_cascade_threshold or true_prob <= neg_cascade_threshold:
        high_conf_idxs.add(idx_i)
        proxy_outputs[idx_i] = True if true_prob >= pos_cascade_threshold else False
```

- **High confidence positive** (score >= tau_pos): accept, đánh dấu True
- **High confidence negative** (score <= tau_neg): reject, đánh dấu False
- **Low confidence** (tau_neg < score < tau_pos): gửi đến oracle LM — `sem_filter.py:507-522`

---

## 3. sem_join Cascade — `sem_join.py:180-333`

### Luồng xử lý

#### Bước 1: join_optimizer — `sem_join.py:242-257`
Gọi `join_optimizer` để chọn strategy tốt nhất (xem C2 chi tiết).

#### Bước 2: run_sem_sim_join làm helper — `sem_join.py:336-366`
```python
def run_sem_sim_join(l1, l2, col1_label, col2_label):
    l2_df = l2.to_frame(name=col2_label)
    l2_df = l2_df.sem_index(col2_label, f"{col2_label}_index")
    K = len(l2)
    out = l1_df.sem_sim_join(l2_df, left_on=col1_label, right_on=col2_label, K=K)
    out["_scores"] = calibrate_sem_sim_join(out["_scores"].tolist())
    return out
```
- Index l2 (embedding), rồi tìm K nearest neighbors cho mỗi item trong l1
- `calibrate_sem_sim_join` — `cascade_utils.py:147-149` — clip scores về [0, 1]

#### Bước 3: learn_join_cascade_threshold — `sem_join.py:530-603`
- Sample từ helper join results — `sem_join.py:564-571`
- Chạy oracle LM trên sample — `sem_join.py:577-587`
- Tìm optimal thresholds qua `learn_cascade_thresholds` — `sem_join.py:589-594`
- Fallback: nếu lỗi, trả về (1.0, 0.0) — full join — `sem_join.py:598-601`

#### Bước 4: Route to oracle — `sem_join.py:266-312`
- High confidence → accept trực tiếp — `sem_join.py:267`
- Low confidence → chạy `sem_filter` với oracle LM — `sem_join.py:291-301`

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

### Khác biệt với filter/join cascade

TopK cascade **không học thresholds trước**. Thay vào đó, `cascade_threshold` là tham số user cung cấp trực tiếp.

### compare_batch_binary_cascade — `sem_topk.py:176-273`

```python
def compare_batch_binary_cascade(pairs, model, user_instruction, cascade_threshold, strategy=None):
```

**Luồng xử lý:**

1. **Tạo prompts** cho tất cả pairs — `sem_topk.py:219-223`

2. **Chạy helper_lm với logprobs** — `sem_topk.py:231`:
   ```python
   helper_output = helper_lm(match_prompts, kwargs={"logprobs": True})
   ```

3. **Format logprobs** — `sem_topk.py:235`:
   ```python
   formatted_logprobs = helper_lm.format_logprobs_for_cascade(helper_logprobs)
   ```
   - `format_logprobs_for_cascade` tại `lm.py:507-515` — extract tokens và confidence values

4. **Check confidence** — `sem_topk.py:247-252`:
   ```python
   for idx_j in range(len(helper_tokens[idx]) - 1, -1, -1):
       if helper_tokens[idx][idx_j].strip(" \n").isnumeric():
           conf = helper_confidences[idx][idx_j]
           if conf >= cascade_threshold:
               high_conf_idxs.add(idx)
   ```
   - Tìm token số (1 hoặc 2) từ cuối lên
   - Nếu confidence của token đó >= threshold → accept helper result

5. **Gửi low-confidence đến large LM** — `sem_topk.py:257-272`:
   ```python
   if len(high_conf_idxs) != len(helper_logprobs):
       low_conf_idxs = sorted([i for i in range(len(helper_logprobs)) if i not in high_conf_idxs])
       large_lm_results = model(large_match_prompts)
       for idx, res in enumerate(large_lm_results.outputs):
           parsed_results[low_conf_idxs[idx]] = parse_ans_binary(res)
   ```

### Sử dụng trong quicksort — `sem_topk.py:437-455`

Cascade được áp dụng trong mỗi partition step của quicksort:
```python
# sem_topk.py:438-444
comparisons, explanations, small_tokens, large_tokens, num_large_calls = compare_batch_binary_cascade(
    pairs, model, user_instruction, cascade_threshold, strategy=strategy
)
```

---

## 5. Cơ sở Toán học

### UB/LB Bounds — `cascade_utils.py:52-56`

```python
def UB(mean, std_dev, s, delta):
    return mean + (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5)

def LB(mean, std_dev, s, delta):
    return mean - (std_dev / (s**0.5)) * ((2 * np.log(1 / delta)) ** 0.5)
```

Đây là **Hoeffding-style concentration inequalities**:
- Với xác suất >= 1 - delta:
  - Giá trị thực <= UB(sample_mean, sample_std, sample_size, delta)
  - Giá trị thực >= LB(sample_mean, sample_std, sample_size, delta)
- `s` = sample_size, `delta` = failure_probability / 2

### Corrected Recall Target — `cascade_utils.py:118-121`

```python
corrected_recall_target = ub_z1 / (ub_z1 + lb_z2)
```

- Z1 = weighted correct trong nhóm scores >= tau_neg (recall numerator)
- Z2 = weighted correct trong nhóm scores < tau_neg (missed items)
- UB(Z1) là giới hạn trên của recall numerator
- LB(Z2) là giới hạn dưới của missed items
- Corrected target đảm bảo: với prob >= 1 - delta, recall thực tế >= recall_target

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

- Cho mỗi candidate tau_pos, tính LB của precision của nhóm accepted
- Dùng Bonferroni correction: delta / len(sorted_pairs) cho mỗi test
- Chọn threshold nhỏ nhất (accept nhiều nhất) mà vẫn đảm bảo precision

---

## 6. So sánh các Cascade approaches

| Khía cạnh | sem_filter | sem_join | sem_topk |
|-----------|-----------|----------|----------|
| Proxy model | Helper LM logprobs HOẶC embedding similarity | Embedding similarity (sem_sim_join) | Helper LM logprobs |
| Threshold learning | Có, từ sample | Có, từ sample | Không, user cung cấp |
| Statistical guarantees | Có (recall + precision) | Có (recall + precision) | Không |
| Khi nào kích hoạt | `cascade_args != None` | `cascade_args != None` và `num_pairs >= min_join_cascade_size` | `cascade_threshold != None` |
| File chính | `sem_filter.py:383-530` | `sem_join.py:180-333` | `sem_topk.py:176-273` |
| Complexity | O(n) proxy + O(sample) oracle learning + O(low_conf) oracle | O(n*m) embedding + O(sample) oracle learning + O(low_conf) oracle | O(n*logn) helper calls + O(low_conf_per_partition) oracle |
