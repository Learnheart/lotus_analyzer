# C5 - Batching Strategies trong LOTUS

## Tong quan

LOTUS su dung batching o hai tang: LM (language model) va RM (retrieval/embedding model).
Batching cho phep gui nhieu requests dong thoi, tang throughput va giam overhead.

---

## 1. LM Batching

### 1.1 Default Batching (khong rate/tpm limit)

**Vi tri:** `lm.py:250-253`

```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True, max_workers=self.max_batch_size, **all_kwargs
)
```

- Su dung `litellm.batch_completion` — import tai `lm.py:10`
- `max_workers=self.max_batch_size` — concurrent threads = batch size
- Default `max_batch_size=64` — `lm.py:73`
- `drop_params=True` — litellm tu dong bo cac params khong duoc model ho tro
- Gui **tat ca uncached messages** trong mot batch call
- Progress bar update sau khi batch hoan tat — `lm.py:253`

### 1.2 Rate-Limited Batching

**Vi tri:** `lm.py:258-303` — `_process_with_rate_limiting`

**Khi nao kich hoat:** `self.rate_limit is not None` — `lm.py:247`

**Cach hoat dong:**
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

**Chi tiet:**
- `max_batch_size` bi cap boi `min(rate_limit, max_batch_size)` — `lm.py:108`
- Moi sub-batch gui dong thoi qua `batch_completion`
- Delay giua batches = `required_time - actual_elapsed` — `lm.py:300-302`
- Khong sleep sau batch cuoi — `lm.py:299`

**Vi du:** rate_limit=60 (60 RPM), max_batch_size=64:
- max_batch_size bi cap = min(60, 64) = 60
- Moi batch: 60 requests dong thoi
- Delay giua batches: 60 seconds - elapsed time

### 1.3 TPM-Limited Batching

**Vi tri:** `lm.py:311-390` — `_process_with_tpm_limiting`

**Khi nao kich hoat:** `self.tpm_limit is not None` — `lm.py:245`

**Cach hoat dong:**

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

3. **Token usage tracking** voi 60-second window — `lm.py:305-309`:
   ```python
   def _get_tokens_used_in_last_minute(self):
       current_time = time.time()
       while self._token_usage_history and self._token_usage_history[0][0] < current_time - 60:
           self._token_usage_history.popleft()  # Evict old entries
       return sum(tokens for _, tokens in self._token_usage_history)
   ```
   - `_token_usage_history` la `deque` — `lm.py:103`
   - Moi entry: `(timestamp, actual_tokens_used)`

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
   - Tinh `available_tokens` = tpm_limit * 0.95 - tokens used in last minute
   - Them requests vao sub-batch cho den khi het budget
   - `effective_max_batch` = `self.max_batch_size` — `lm.py:344`

5. **Record actual usage** — `lm.py:367-369`:
   ```python
   actual_tokens = sum(r.usage.total_tokens for r in sub_responses if hasattr(r, "usage"))
   self._token_usage_history.append((start_time, actual_tokens))
   ```

6. **Wait khi het token budget** — `lm.py:380-388`:
   ```python
   else:  # sub_batch is empty
       wait_time = 1.0
       if self._token_usage_history:
           wait_time = max(0.1, self._token_usage_history[0][0] + 60.1 - time.time())
       pbar.set_postfix_str(f"TPM Limit reached, waiting {wait_time:.1f}s")
       time.sleep(wait_time)
   ```
   - Tinh thoi gian cho den khi entry cu nhat expire (> 60s)
   - Hien thi "TPM Limit reached" tren progress bar

7. **RPM enforcement** khi co ca rate_limit — `lm.py:374-379`:
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
- Dung `SentenceTransformer.encode()` — GPU-accelerated
- `convert_to_base_data` xu ly ImageDtype → base types

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
- Optional `truncate_limit` de cat text qua dai — `litellm_rm.py:29`
- Dung `litellm.embedding()` — API call

---

## 3. Batching trong Operators

### 3.1 sem_filter — batch LM calls
Tat ca docs duoc tao prompts, roi gui ca batch den `model(inputs)` — `sem_filter.py:112-114`.
LM noi bo xu ly batching.

### 3.2 sem_join — batch per left element
Tat ca pairs (l1 x l2) duoc tao docs, roi gui mot batch duy nhat — `sem_join.py:128-147`.
So pairs = len(l1) * len(l2).

### 3.3 sem_topk — batch per partition
Moi partition step trong quicksort tao pairs roi gui batch — `sem_topk.py:426-428`.
So pairs moi partition = high - low (co the lon).

### 3.4 sem_agg — batch per tree level
Moi level cua aggregation tree tao batch prompts — `sem_agg.py:211`.
So prompts moi level giam theo context capacity.

---

## 4. Bang tom tat

| Component | Default batch size | Batching mechanism | Rate control |
|-----------|-------------------|-------------------|--------------|
| LM (default) | 64 | `litellm.batch_completion` | Khong |
| LM (rate limited) | min(rate_limit, 64) | Sub-batches + sleep | RPM |
| LM (TPM limited) | 64 (cap boi token budget) | Dynamic sub-batches | TPM + optional RPM |
| SentenceTransformersRM | 64 | Loop + `encoder.encode()` | Khong |
| LiteLLMRM | 64 | Loop + `litellm.embedding()` | Khong |
