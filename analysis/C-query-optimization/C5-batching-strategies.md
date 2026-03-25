# C5 - Batching Strategies trong LOTUS

## Tổng quan

LOTUS sử dụng batching ở hai tầng: LM (language model) và RM (retrieval/embedding model).
Batching cho phép gửi nhiều requests đồng thời, tăng throughput và giảm overhead.

---

## 1. LM Batching

### 1.1 Default Batching (không rate/tpm limit)

**Vị trí:** `lm.py:250-253`

```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True, max_workers=self.max_batch_size, **all_kwargs
)
```

- Sử dụng `litellm.batch_completion` — import tại `lm.py:10`
- `max_workers=self.max_batch_size` — concurrent threads = batch size
- Default `max_batch_size=64` — `lm.py:73`
- `drop_params=True` — litellm tự động bỏ các params không được model hỗ trợ
- Gửi **tất cả uncached messages** trong một batch call
- Progress bar update sau khi batch hoàn tất — `lm.py:253`

### 1.2 Rate-Limited Batching

**Vị trí:** `lm.py:258-303` — `_process_with_rate_limiting`

**Khi nào kích hoạt:** `self.rate_limit is not None` — `lm.py:247`

**Cách hoạt động:**
```python
# lm.py:276-303
num_batches = math.ceil(len(batch) / self.max_batch_size)
min_interval_per_request = 60 / self.rate_limit  # seconds/request

for i in range(num_batches):
    start_time = time.time()
    sub_batch = batch[start_idx:end_idx]
    sub_responses = batch_completion(self.model, sub_batch, ...)

    # Calculate required delay
    required_time_for_batch = len(sub_batch) * min_interval_per_request
    if i < num_batches - 1:  # Don't sleep after last batch
        to_sleep = required_time_for_batch - elapsed
        if to_sleep > 0:
            time.sleep(to_sleep)
```

**Chi tiết:**
- `max_batch_size` bị cấp bởi `min(rate_limit, max_batch_size)` — `lm.py:108`
- Mỗi sub-batch gửi đồng thời qua `batch_completion`
- Delay giữa batches = `required_time - actual_elapsed` — `lm.py:300-302`
- Không sleep sau batch cuối — `lm.py:299`

**Ví dụ:** rate_limit=60 (60 RPM), max_batch_size=64:
- max_batch_size bị cấp = min(60, 64) = 60
- Mỗi batch: 60 requests đồng thời
- Delay giữa batches: 60 seconds - elapsed time

### 1.3 TPM-Limited Batching

**Vị trí:** `lm.py:311-390` — `_process_with_tpm_limiting`

**Khi nào kích hoạt:** `self.tpm_limit is not None` — `lm.py:245`

**Cách hoạt động:**

1. **95% Safety buffer** — `lm.py:317`:
   ```python
   max_allowed_tpm = int(self.tpm_limit * 0.95)
   ```

2. **Pre-compute token estimates** — `lm.py:319-330`:
   ```python
   for idx, msg in enumerate(batch):
       est = self.count_tokens(msg)
       total_est = est + self.max_tokens  # Input + expected output
       if total_est > max_allowed_tpm:
           raise ValueError(f"Row {idx} too large for TPM limit")
       token_estimates.append(est)
   ```

3. **Token usage tracking** với 60-second window — `lm.py:305-309`:
   ```python
   def _get_tokens_used_in_last_minute(self):
       current_time = time.time()
       while self._token_usage_history and self._token_usage_history[0][0] < current_time - 60:
           self._token_usage_history.popleft()  # Evict old entries
       return sum(tokens for _, tokens in self._token_usage_history)
   ```
   - `_token_usage_history` là `deque` — `lm.py:103`
   - Mỗi entry: `(timestamp, actual_tokens_used)`

4. **Dynamic sub-batch sizing** — `lm.py:332-358`:
   ```python
   while i < len(batch):
       available_tokens = max(0, int(self.tpm_limit * 0.95) - tokens_in_last_minute)
       sub_batch = []
       sub_batch_estimate = 0

       while i < len(batch):
           est = token_estimates[i] + self.max_tokens
           if sub_batch_estimate + est <= available_tokens:
               sub_batch.append(batch[i])
               sub_batch_estimate += est
               i += 1
               if len(sub_batch) >= effective_max_batch:
                   break
           else:
               break
   ```
   - Tính `available_tokens` = tpm_limit * 0.95 - tokens used in last minute
   - Thêm requests vào sub-batch cho đến khi hết budget
   - `effective_max_batch` = `self.max_batch_size` — `lm.py:344`

5. **Record actual usage** — `lm.py:367-369`:
   ```python
   actual_tokens = sum(r.usage.total_tokens for r in sub_responses if hasattr(r, "usage"))
   self._token_usage_history.append((start_time, actual_tokens))
   ```

6. **Wait khi hết token budget** — `lm.py:380-388`:
   ```python
   else:  # sub_batch is empty
       wait_time = 1.0
       if self._token_usage_history:
           wait_time = max(0.1, self._token_usage_history[0][0] + 60.1 - time.time())
       pbar.set_postfix_str(f"TPM Limit reached, waiting {wait_time:.1f}s")
       time.sleep(wait_time)
   ```
   - Tính thời gian chờ đến khi entry cũ nhất expire (> 60s)
   - Hiển thị "TPM Limit reached" trên progress bar

7. **RPM enforcement** khi có cả rate_limit — `lm.py:374-379`:
   ```python
   if self.rate_limit is not None:
       elapsed = time.time() - start_time
       required_time_for_rpm = len(sub_batch) * (60 / self.rate_limit)
       to_sleep = required_time_for_rpm - elapsed
       if to_sleep > 0:
           time.sleep(to_sleep)
   ```

---

## 2. Embedding Batching

### 2.1 SentenceTransformersRM — `sentence_transformers_rm.py:49-76`

```python
def _embed(self, docs):
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):
        batch = docs[i : i + self.max_batch_size]
        _batch = convert_to_base_data(batch)
        torch_embeddings = self.transformer.encode(
            _batch, convert_to_tensor=True,
            normalize_embeddings=self.normalize_embeddings,
            show_progress_bar=False
        )
        cpu_embeddings = torch_embeddings.cpu().numpy()
        all_embeddings.append(cpu_embeddings)
    return np.vstack(all_embeddings)
```

- Default `max_batch_size=64` — `sentence_transformers_rm.py:29`
- Dùng `SentenceTransformer.encode()` — GPU-accelerated
- `convert_to_base_data` xử lý ImageDtype → base types

### 2.2 LiteLLMRM — `litellm_rm.py:45-71`

```python
def _embed(self, docs):
    all_embeddings = []
    for i in tqdm(range(0, len(docs), self.max_batch_size)):
        batch = docs[i : i + self.max_batch_size]
        if self.truncate_limit:
            batch = [doc[:self.truncate_limit] for doc in batch]
        _batch = convert_to_base_data(batch)
        response = embedding(model=self.model, input=_batch)
        embeddings = np.array([d["embedding"] for d in response.data])
        all_embeddings.append(embeddings)
    return np.vstack(all_embeddings)
```

- Default `max_batch_size=64` — `litellm_rm.py:28`
- Optional `truncate_limit` để cắt text quá dài — `litellm_rm.py:29`
- Dùng `litellm.embedding()` — API call

---

## 3. Batching trong Operators

### 3.1 sem_filter — batch LM calls
Tất cả docs được tạo prompts, rồi gửi cả batch đến `model(inputs)` — `sem_filter.py:112-114`.
LM nội bộ xử lý batching.

### 3.2 sem_join — batch per left element
Tất cả pairs (l1 x l2) được tạo docs, rồi gửi một batch duy nhất — `sem_join.py:128-147`.
Số pairs = len(l1) * len(l2).

### 3.3 sem_topk — batch per partition
Mỗi partition step trong quicksort tạo pairs rồi gửi batch — `sem_topk.py:426-428`.
Số pairs mỗi partition = high - low (có thể lớn).

### 3.4 sem_agg — batch per tree level
Mỗi level của aggregation tree tạo batch prompts — `sem_agg.py:211`.
Số prompts mỗi level giảm theo context capacity.

---

## 4. Bảng tóm tắt

| Component | Default batch size | Batching mechanism | Rate control |
|-----------|-------------------|-------------------|--------------|
| LM (default) | 64 | `litellm.batch_completion` | Không |
| LM (rate limited) | min(rate_limit, 64) | Sub-batches + sleep | RPM |
| LM (TPM limited) | 64 (cấp bởi token budget) | Dynamic sub-batches | TPM + optional RPM |
| SentenceTransformersRM | 64 | Loop + `encoder.encode()` | Không |
| LiteLLMRM | 64 | Loop + `litellm.embedding()` | Không |
