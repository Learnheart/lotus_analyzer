# A2 - Batching & Concurrency

## Batching trong LM

### Default batching (không có rate/tpm limit)

Khi không có `rate_limit` hay `tpm_limit`, LM gửi tất cả messages cùng lúc qua `batch_completion` (lm.py:250-251):

```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True, max_workers=self.max_batch_size, **all_kwargs
)
```

- `max_workers=self.max_batch_size`: default 64 (lm.py:73)
- `batch` là list của tất cả uncached messages
- litellm nội bộ sử dụng ThreadPoolExecutor với `max_workers` threads

### max_batch_size default và interaction với rate_limit (lm.py:105-112)

```python
if rate_limit is not None:
    self._rate_limit_delay: float = 60 / rate_limit
    if max_batch_size is not None:
        self.max_batch_size = min(rate_limit, max_batch_size)
    else:
        self.max_batch_size = rate_limit
else:
    self.max_batch_size = max_batch_size
```

Khi `rate_limit` được set, `max_batch_size` bị cap lại để không vượt quá RPM limit.

## Rate Limiting: _process_with_rate_limiting (lm.py:258)

```python
def _process_with_rate_limiting(
    self, batch: list[list[dict[str, str]]], all_kwargs: dict[str, Any], pbar: tqdm
) -> list[ModelResponse]:
```

Thuật toán:
1. Chia `batch` thành sub-batches có kích thước `max_batch_size`
2. Với mỗi sub-batch:
   - Gửi `batch_completion()` (lm.py:286-288)
   - Tính thời gian cần thiết theo RPM: `required_time = len(sub_batch) * (60 / rate_limit)` (lm.py:296)
   - Nếu batch chạy nhanh hơn `required_time`, `sleep()` phần thời gian còn lại (lm.py:300-302)
   - Không sleep sau batch cuối cùng (lm.py:299)

Ví dụ: `rate_limit=100`, `max_batch_size=64`
- 200 messages -> 4 sub-batches (64, 64, 64, 8)
- Mỗi sub-batch phải mất ít nhất `64 * 0.6s = 38.4s`

## TPM Limiting: _process_with_tpm_limiting (lm.py:311)

```python
def _process_with_tpm_limiting(
    self, batch: list[list[dict[str, str]]], all_kwargs: dict[str, Any], pbar: tqdm
) -> list[ModelResponse]:
```

Đây là cơ chế phức tạp hơn, theo dõi token usage trong sliding window 1 phút.

### Thuật toán chi tiết:

1. **Ước tính tokens** (lm.py:319-330):
   - Mỗi message được ước tính: `est = count_tokens(msg) + max_tokens`
   - Kiểm tra mỗi message không vượt quá 95% TPM limit
   - Nếu vượt -> raise `ValueError` (lm.py:324-329)

2. **Sliding window tracking** (lm.py:305-309):
   ```python
   def _get_tokens_used_in_last_minute(self) -> int:
       current_time = time.time()
       while self._token_usage_history and self._token_usage_history[0][0] < current_time - 60:
           self._token_usage_history.popleft()
       return sum(tokens for _, tokens in self._token_usage_history)
   ```
   Sử dụng `deque` (lm.py:103) để lưu (timestamp, tokens) tuples. Xóa entries cũ hơn 60s.

3. **Build sub-batch** (lm.py:339-358):
   - Tính `available_tokens = 0.95 * tpm_limit - tokens_used_in_last_minute`
   - Thêm messages vào sub-batch cho đến khi hết budget hoặc đạt `effective_max_batch`
   - Nếu không thêm được message nào (token budget = 0), wait cho window clear (lm.py:381-388)

4. **Record actual usage** (lm.py:368-369):
   ```python
   actual_tokens = sum(r.usage.total_tokens for r in sub_responses if hasattr(r, "usage"))
   self._token_usage_history.append((start_time, actual_tokens))
   ```

5. **Combined RPM+TPM** (lm.py:374-379):
   Nếu cả `rate_limit` và `tpm_limit` đều được set, sau mỗi sub-batch còn enforce RPM delay.

### Safety buffer: 5% (lm.py:317, lm.py:336)
```python
max_allowed_tpm = int(self.tpm_limit * 0.95)
```

## Progress Bar

Tất cả processing methods sử dụng `tqdm` progress bar (lm.py:236-241):

```python
pbar = tqdm(
    total=total_calls,
    desc=progress_bar_desc,
    disable=not show_progress_bar,
    bar_format="{l_bar}{bar} {n}/{total} LM calls [{elapsed}<{remaining}, {rate_fmt}{postfix}]",
)
```

- Default `show_progress_bar=True`
- TPM limiting hiển thị trạng thái wait: `pbar.set_postfix_str(f"TPM Limit reached, waiting {wait_time:.1f}s")` (lm.py:386)

## Concurrency trong Operators

### Group-by parallelism

`sem_agg` (sem_agg.py:396-399) và `sem_topk` (sem_topk.py:770-773) sử dụng `ThreadPoolExecutor` cho group_by:

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

- Default 8 threads (settings.py:23)
- Mỗi group chạy `sem_agg`/`sem_topk` độc lập

### llm_as_judge parallelism (llm_as_judge.py:83-103)

```python
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    sem_map_outputs = list(
        executor.map(
            lambda _: sem_map(...),
            range(n_trials),
        )
    )
```

- Chạy n_trials `sem_map` calls song song
- Lưu ý: `lotus.settings.enable_cache = False` trước khi chạy (llm_as_judge.py:82)
