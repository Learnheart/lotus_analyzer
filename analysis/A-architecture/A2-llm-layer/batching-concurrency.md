# A2 - Batching & Concurrency

## Batching trong LM

### Default batching (khong co rate/tpm limit)

Khi khong co `rate_limit` hay `tpm_limit`, LM gui tat ca messages cung luc qua `batch_completion` (lm.py:250-251):

```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True, max_workers=self.max_batch_size, **all_kwargs
)
```

- `max_workers=self.max_batch_size`: default 64 (lm.py:73)
- `batch` la list cua tat ca uncached messages
- litellm noi bo su dung ThreadPoolExecutor voi `max_workers` threads

### max_batch_size default va interaction voi rate_limit (lm.py:105-112)

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

Khi `rate_limit` duoc set, `max_batch_size` bi cap lai de khong vuot qua RPM limit.

## Rate Limiting: _process_with_rate_limiting (lm.py:258)

```python
def _process_with_rate_limiting(
    self, batch: list[list[dict[str, str]]], all_kwargs: dict[str, Any], pbar: tqdm
) -> list[ModelResponse]:
```

Thuat toan:
1. Chia `batch` thanh sub-batches co kich thuoc `max_batch_size`
2. Voi moi sub-batch:
   - Gui `batch_completion()` (lm.py:286-288)
   - Tinh thoi gian can thiet theo RPM: `required_time = len(sub_batch) * (60 / rate_limit)` (lm.py:296)
   - Neu batch chay nhanh hon `required_time`, `sleep()` phan thoi gian con lai (lm.py:300-302)
   - Khong sleep sau batch cuoi cung (lm.py:299)

Vi du: `rate_limit=100`, `max_batch_size=64`
- 200 messages -> 4 sub-batches (64, 64, 64, 8)
- Moi sub-batch phai mat it nhat `64 * 0.6s = 38.4s`

## TPM Limiting: _process_with_tpm_limiting (lm.py:311)

```python
def _process_with_tpm_limiting(
    self, batch: list[list[dict[str, str]]], all_kwargs: dict[str, Any], pbar: tqdm
) -> list[ModelResponse]:
```

Day la co che phuc tap hon, theo doi token usage trong sliding window 1 phut.

### Thuat toan chi tiet:

1. **Uoc tinh tokens** (lm.py:319-330):
   - Moi message duoc uoc tinh: `est = count_tokens(msg) + max_tokens`
   - Kiem tra moi message khong vuot qua 95% TPM limit
   - Neu vuot -> raise `ValueError` (lm.py:324-329)

2. **Sliding window tracking** (lm.py:305-309):
   ```python
   def _get_tokens_used_in_last_minute(self) -> int:
       current_time = time.time()
       while self._token_usage_history and self._token_usage_history[0][0] < current_time - 60:
           self._token_usage_history.popleft()
       return sum(tokens for _, tokens in self._token_usage_history)
   ```
   Su dung `deque` (lm.py:103) de luu (timestamp, tokens) tuples. Xoa entries cu hon 60s.

3. **Build sub-batch** (lm.py:339-358):
   - Tinh `available_tokens = 0.95 * tpm_limit - tokens_used_in_last_minute`
   - Them messages vao sub-batch cho den khi het budget hoac dat `effective_max_batch`
   - Neu khong them duoc message nao (token budget = 0), wait cho window clear (lm.py:381-388)

4. **Record actual usage** (lm.py:368-369):
   ```python
   actual_tokens = sum(r.usage.total_tokens for r in sub_responses if hasattr(r, "usage"))
   self._token_usage_history.append((start_time, actual_tokens))
   ```

5. **Combined RPM+TPM** (lm.py:374-379):
   Neu ca `rate_limit` va `tpm_limit` deu duoc set, sau moi sub-batch con enforce RPM delay.

### Safety buffer: 5% (lm.py:317, lm.py:336)
```python
max_allowed_tpm = int(self.tpm_limit * 0.95)
```

## Progress Bar

Tat ca processing methods su dung `tqdm` progress bar (lm.py:236-241):

```python
pbar = tqdm(
    total=total_calls,
    desc=progress_bar_desc,
    disable=not show_progress_bar,
    bar_format="{l_bar}{bar} {n}/{total} LM calls [{elapsed}<{remaining}, {rate_fmt}{postfix}]",
)
```

- Default `show_progress_bar=True`
- TPM limiting hien thi trang thai wait: `pbar.set_postfix_str(f"TPM Limit reached, waiting {wait_time:.1f}s")` (lm.py:386)

## Concurrency trong Operators

### Group-by parallelism

`sem_agg` (sem_agg.py:396-399) va `sem_topk` (sem_topk.py:770-773) su dung `ThreadPoolExecutor` cho group_by:

```python
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

- Default 8 threads (settings.py:23)
- Moi group chay `sem_agg`/`sem_topk` doc lap

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

- Chay n_trials `sem_map` calls song song
- Luu y: `lotus.settings.enable_cache = False` truoc khi chay (llm_as_judge.py:82)
