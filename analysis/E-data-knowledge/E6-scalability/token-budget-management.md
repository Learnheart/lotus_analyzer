# E6 - Token Budget Management

## Tổng quan

LOTUS cung cấp UsageLimit cho token/cost control, LM tracks virtual vs physical usage, và safe_mode cho cost estimation trước execution.

---

## 1. UsageLimit Configuration

**Location**: `types.py:216-220`

```python
@dataclass
class UsageLimit:
    prompt_tokens_limit: float = float("inf")       # :217
    completion_tokens_limit: float = float("inf")    # :218
    total_tokens_limit: float = float("inf")         # :219
    total_cost_limit: float = float("inf")           # :220
```

Tất cả default `inf` → không giới hạn trừ khi user set.

### Configuration
```python
lm = LM(
    model="gpt-4o-mini",
    physical_usage_limit=UsageLimit(total_cost_limit=10.0),   # $10 max
    virtual_usage_limit=UsageLimit(total_tokens_limit=1000000)  # 1M tokens max
)
```

---

## 2. Usage Limit Checking

**Location**: `lm.py:419-427`

```python
def _check_usage_limit(self, usage, limit, usage_type):
    if (
        usage.prompt_tokens > limit.prompt_tokens_limit         # :421
        or usage.completion_tokens > limit.completion_tokens_limit  # :422
        or usage.total_tokens > limit.total_tokens_limit        # :423
        or usage.total_cost > limit.total_cost_limit            # :424
    ):
        raise LotusUsageLimitException(                         # :426
            f"Usage limit exceeded. Current {usage_type} usage: {usage}, Limit: {limit}"
        )
```

Check xảy ra sau mỗi response (`lm.py:478, 483`):
```python
self._check_usage_limit(self.stats.virtual_usage, self.virtual_usage_limit, "virtual")   # :478
self._check_usage_limit(self.stats.physical_usage, self.physical_usage_limit, "physical") # :483
```

Exception `LotusUsageLimitException` (`types.py:232-234`) dừng execution khi vượt limit.

---

## 3. Virtual vs Physical Usage

**Location**: `lm.py:451-483`

```python
def _update_stats(self, response, is_cached=False):
    # Always update virtual usage
    self._update_usage_stats(self.stats.virtual_usage, response, cost)   # :477
    self._check_usage_limit(..., "virtual")                               # :478

    # Only update physical for non-cached
    if not is_cached:                                                     # :481
        self._update_usage_stats(self.stats.physical_usage, response, cost)  # :482
        self._check_usage_limit(..., "physical")                             # :483
```

### Virtual Usage
- Tổng usage nếu không có caching
- Includes cached responses
- Reflects "true cost" nếu cache bị disable

### Physical Usage
- Actual API calls made
- Excludes cached responses
- Reflects actual billing

### LMStats tracking (`types.py:20-67`)
```python
@dataclass
class LMStats:
    @dataclass
    class TotalUsage:
        prompt_tokens: int = 0
        completion_tokens: int = 0
        total_tokens: int = 0
        total_cost: float = 0.0
        cached_prompt_tokens: int = 0        # :28
        cache_creation_tokens: int = 0       # :30

    virtual_usage: TotalUsage               # :53
    physical_usage: TotalUsage              # :55
    cache_hits: int = 0                     # :57
```

---

## 4. Cost Calculation

**Location**: `lm.py:467-474`

```python
cost = calculate_cost_from_response(response)    # :468
if cost is not None:
    lotus.logger.debug(f"Calculated cost: ${cost:.6f}")
else:
    warnings.warn(f"No pricing information available for model {self.model}")
```

Dùng litellm's pricing database cho cost calculation. Hỗ trợ cached token pricing:

```python
# lm.py:438-449
if hasattr(response.usage, "prompt_tokens_details"):
    cached_tokens = details.get("cached_tokens", 0)          # :442
    cache_creation_tokens = details.get("cache_creation_tokens", 0)  # :443
    usage.cached_prompt_tokens += cached_tokens               # :448
    usage.cache_creation_tokens += cache_creation_tokens      # :449
```

---

## 5. safe_mode Cost Estimation

### sem_filter safe_mode (`sem_filter.py:107-110`)
```python
if safe_mode:
    estimated_total_calls = len(docs)                                        # :108
    estimated_total_cost = sum(model.count_tokens(input) for input in inputs)  # :109
    show_safe_mode(estimated_total_cost, estimated_total_calls)               # :110
```

### show_safe_mode (`utils.py:123-134`)
```python
def show_safe_mode(estimated_cost, estimated_LM_calls):
    print(f"Estimated cost: {estimated_cost} tokens")
    print(f"Estimated LM calls: {estimated_LM_calls}")
    for i in range(5, 0, -1):
        print(f"Proceeding execution in {i} seconds... Press CTRL+C to cancel")
        time.sleep(1)
```

5-second countdown cho user cancel trước execution. Cost estimated in tokens (not dollars).

---

## 6. TPM (Tokens Per Minute) Limiting

**Location**: `lm.py:311-390`

```python
def _process_with_tpm_limiting(self, batch, all_kwargs, pbar):
    available_tokens = max(0, int(self.tpm_limit * 0.95) - tokens_in_last_minute)  # :336
```

- 5% safety buffer (`tpm_limit * 0.95`)
- Adaptive batch sizing based on available tokens
- Auto-wait khi TPM limit reached (`lm.py:381-388`)
- Prevents API rate limit errors

---

## 7. Usage Reporting

**Location**: `lm.py:579-587`

```python
def print_total_usage(self):
    print(f"Virtual Cost:     ${self.stats.virtual_usage.total_cost:,.6f}")
    print(f"Physical Cost:    ${self.stats.physical_usage.total_cost:,.6f}")
    print(f"Virtual Tokens:   {self.stats.virtual_usage.total_tokens:,}")
    print(f"Physical Tokens:  {self.stats.physical_usage.total_tokens:,}")
    print(f"Cache Hits:       {self.stats.cache_hits:,}")
```

---

## 8. Kết luận

Token budget management trong LOTUS:
- **UsageLimit**: Configurable limits cho prompt/completion/total tokens và cost
- **Virtual vs Physical**: Track actual vs would-be usage
- **Cost calculation**: Via litellm pricing database
- **safe_mode**: Pre-execution cost estimation with cancellation
- **TPM limiting**: Automatic token-per-minute rate control
- **Exception-based**: LotusUsageLimitException stops execution
