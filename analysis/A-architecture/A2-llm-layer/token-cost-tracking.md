# A2 - Token & Cost Tracking

## LMStats Dataclass

Định nghĩa tại `types.py:20`:

```python
@dataclass
class LMStats:
    @dataclass
    class TotalUsage:
        prompt_tokens: int = 0
        completion_tokens: int = 0
        total_tokens: int = 0
        total_cost: float = 0.0
        cached_prompt_tokens: int = 0
        cache_creation_tokens: int = 0

    virtual_usage: TotalUsage = field(default_factory=TotalUsage)
    physical_usage: TotalUsage = field(default_factory=TotalUsage)
    cache_hits: int = 0
    operator_cache_hits: int = 0
```

### Hai loại usage:
- **virtual_usage** (types.py:53): Tổng usage NHƯ THỂ không có caching. Cập nhật cho mọi response, kể cả cached. Phản ánh "chi phí thật" nếu không dùng cache.
- **physical_usage** (types.py:55): Usage thực tế với caching. Chỉ cập nhật cho uncached responses.
- **cache_hits** (types.py:57): Số LM responses lấy từ cache.
- **operator_cache_hits** (types.py:58): Số operator results lấy từ operator cache.

### TotalUsage hỗ trợ arithmetic (types.py:32-50):
- `__sub__`: `usage1 - usage2` -> difference
- `__add__`: `usage1 + usage2` -> sum

Được sử dụng trong operator_cache để tính virtual usage delta (cache.py:93).

## Cost Calculation

### pricing.py:10

```python
def calculate_cost_from_response(response) -> Optional[float]:
    try:
        return completion_cost(completion_response=response)
    except litellm.exceptions.NotFoundError as e:
        logger.debug(f"Model pricing not found in LiteLLM: {e}")
        return None
    except Exception as e:
        logger.debug(f"Unexpected error calculating completion cost: {e}")
        return None
```

- Sử dụng `litellm.completion_cost` (pricing.py:5) - database giá được litellm maintain
- Trả về `None` nếu model không có trong pricing database
- Xử lý graceful: không raise exception, chỉ log warning

### _update_stats flow (lm.py:451-483)

```python
def _update_stats(self, response: ModelResponse, is_cached: bool = False):
    cost = calculate_cost_from_response(response)

    if cost is not None:
        lotus.logger.debug(f"Calculated cost: ${cost:.6f} for model {self.model}")
    else:
        lotus.logger.debug(f"No pricing information available for model {self.model}")
        warnings.warn(f"No pricing information available for model {self.model}. ")

    # Always update virtual usage
    self._update_usage_stats(self.stats.virtual_usage, response, cost)
    self._check_usage_limit(self.stats.virtual_usage, self.virtual_usage_limit, "virtual")

    # Only update physical usage for non-cached responses
    if not is_cached:
        self._update_usage_stats(self.stats.physical_usage, response, cost)
        self._check_usage_limit(self.stats.physical_usage, self.physical_usage_limit, "physical")
```

### _update_usage_stats (lm.py:429-449)

```python
def _update_usage_stats(self, usage: LMStats.TotalUsage, response: ModelResponse, cost: float | None):
    if hasattr(response, "usage") and response.usage:
        usage.prompt_tokens += response.usage.prompt_tokens or 0
        usage.completion_tokens += response.usage.completion_tokens or 0
        usage.total_tokens += response.usage.total_tokens or 0
        if cost is not None:
            usage.total_cost += cost

        # Extract cached token information
        if hasattr(response.usage, "prompt_tokens_details") and response.usage.prompt_tokens_details:
            details = response.usage.prompt_tokens_details
            # Handle both dict and object formats
            cached_tokens = ...
            cache_creation_tokens = ...
            usage.cached_prompt_tokens += cached_tokens
            usage.cache_creation_tokens += cache_creation_tokens
```

Đặc biệt theo dõi `cached_prompt_tokens` và `cache_creation_tokens` từ `prompt_tokens_details` (lm.py:438-449). Đây là provider-level prompt caching (e.g. OpenAI), không phải LOTUS cache.

## Usage Limits

### UsageLimit dataclass (types.py:216)

```python
@dataclass
class UsageLimit:
    prompt_tokens_limit: float = float("inf")
    completion_tokens_limit: float = float("inf")
    total_tokens_limit: float = float("inf")
    total_cost_limit: float = float("inf")
```

Default: không có giới hạn (infinity).

### _check_usage_limit (lm.py:419-427)

```python
def _check_usage_limit(self, usage: LMStats.TotalUsage, limit: UsageLimit, usage_type: str):
    if (
        usage.prompt_tokens > limit.prompt_tokens_limit
        or usage.completion_tokens > limit.completion_tokens_limit
        or usage.total_tokens > limit.total_tokens_limit
        or usage.total_cost > limit.total_cost_limit
    ):
        raise LotusUsageLimitException(f"Usage limit exceeded. Current {usage_type} usage: {usage}, Limit: {limit}")
```

- Kiểm tra sau mỗi response update
- Raise `LotusUsageLimitException` (types.py:232) khi vượt bất kỳ limit nào
- Kiểm tra riêng cho virtual và physical usage

### Cách sử dụng:

```python
lm = LM(
    model="gpt-4o-mini",
    physical_usage_limit=UsageLimit(total_cost_limit=10.0),   # max $10
    virtual_usage_limit=UsageLimit(total_tokens_limit=1000000), # max 1M tokens
)
```

## Utility Methods

### print_total_usage (lm.py:579-587)

```python
def print_total_usage(self):
    print("\n=== Usage Statistics ===")
    print("Virtual  = Total usage if no caching was used")
    print("Physical = Actual usage with caching applied\n")
    print(f"Virtual Cost:     ${self.stats.virtual_usage.total_cost:,.6f}")
    print(f"Physical Cost:    ${self.stats.physical_usage.total_cost:,.6f}")
    print(f"Virtual Tokens:   {self.stats.virtual_usage.total_tokens:,}")
    print(f"Physical Tokens:  {self.stats.physical_usage.total_tokens:,}")
    print(f"Cache Hits:       {self.stats.cache_hits:,}\n")
```

### reset_stats (lm.py:589-590)

```python
def reset_stats(self):
    self.stats = LMStats()
```

Tạo mới LMStats instance, reset tất cả counters.

## Safe Mode

Một số operators hỗ trợ `safe_mode` parameter. Khi bật, operator ước tính token usage trước khi gọi LM:

```python
# sem_filter.py:107-110
if safe_mode:
    estimated_total_calls = len(docs)
    estimated_total_cost = sum(model.count_tokens(input) for input in inputs)
    show_safe_mode(estimated_total_cost, estimated_total_calls)
```

Và in usage thực tế sau khi hoàn thành:
```python
# sem_filter.py:121-122
if safe_mode:
    model.print_total_usage()
```
