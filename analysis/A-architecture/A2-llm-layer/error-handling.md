# A2 - Error Handling

## Custom Exceptions

Định nghĩa tại `types.py:226-235`:

```python
class LotusException(Exception):
    """Base class for all Lotus exceptions."""
    pass

class LotusUsageLimitException(LotusException):
    """Exception raised when the usage limit is exceeded."""
    pass
```

### Hierarchy:
```
Exception
  |-- LotusException (types.py:226)
        |-- LotusUsageLimitException (types.py:232)
```

## LM Error Handling

### Response error detection (lm.py:485-494)

```python
def _get_top_choice(self, response: ModelResponse) -> str:
    # Handle authentication errors and other exceptions
    if isinstance(response, (AuthenticationError, OpenAIError)):
        raise response

    choice = response.choices[0]
    assert isinstance(choice, Choices)
    if choice.message.content is None:
        raise ValueError(f"No content in response: {response}")
    return choice.message.content
```

`litellm.batch_completion` có thể trả về exception objects thay vì `ModelResponse` khi lỗi xảy ra. LM kiểm tra `isinstance(response, (AuthenticationError, OpenAIError))` và re-raise chúng.

- `AuthenticationError`: import từ `litellm.exceptions` (lm.py:11)
- `OpenAIError`: import từ `openai._exceptions` (lm.py:14)

### Tương tự cho logprobs (lm.py:496-505):

```python
def _get_top_choice_logprobs(self, response: ModelResponse) -> list[ChatCompletionTokenLogprob]:
    if isinstance(response, (AuthenticationError, OpenAIError)):
        raise response
    ...
```

### Cache error handling (lm.py:392-405)

```python
def _cache_response(self, response: ModelResponse, hash: str) -> None:
    if isinstance(response, OpenAIError):
        raise response
    self.cache.insert(hash, response)
```

Không cache error responses, raise chúng thay vì lưu vào cache.

### Usage limit check (lm.py:419-427)

```python
def _check_usage_limit(self, usage: LMStats.TotalUsage, limit: UsageLimit, usage_type: str):
    if (
        usage.prompt_tokens > limit.prompt_tokens_limit
        or usage.completion_tokens > limit.completion_tokens_limit
        or usage.total_tokens > limit.total_tokens_limit
        or usage.total_cost > limit.total_cost_limit
    ):
        raise LotusUsageLimitException(...)
```

Được gọi sau mỗi `_update_stats` (lm.py:478, 483). Kiểm tra cả virtual và physical limits.

### TPM limit validation (lm.py:323-329)

```python
if total_est > max_allowed_tpm:
    raise ValueError(
        f"Row {idx} estimated size ({total_est} tokens) exceeds your "
        f"total TPM limit with safety buffer ({max_allowed_tpm} tokens). "
        f"This row is too large to ever be sent on your current API tier. "
        f"Please trim your data or upgrade your account tier."
    )
```

Pre-check trước khi gửi request: nếu 1 row có token count > 95% TPM limit, không có cách nào gửi được.

## Retry Mechanism

LOTUS phụ thuộc vào `backoff` package (được list trong dependencies) cho retry logic. Tuy nhiên, retry được implement bởi `litellm` bên trong `batch_completion`, không phải bởi LOTUS code trực tiếp.

## Operator-level Error Handling

### Settings validation

Mỗi operator kiểm tra model configuration:

```python
# sem_filter.py:351-354
if lotus.settings.lm is None:
    raise ValueError(
        "The language model must be an instance of LM. Please configure a valid language model using lotus.settings.configure()"
    )
```

Tương tự cho RM/VS operators:
```python
# sem_search.py:106-109
if rm is None or vs is None:
    raise ValueError(
        "The retrieval model must be an instance of RM, and the vector store should be an instance of VS..."
    )
```

### Column validation

```python
# sem_filter.py:363-365
for column in col_li:
    if column not in self._obj.columns:
        raise ValueError(f"Column {column} not found in DataFrame")
```

### nl_expression parse error (nl_expression.py:10-13)

```python
if not matches:
    raise ValueError(
        "Language expression contains no parameterized columns. Please specify the name of the relevant data column(s) in brackets {} within your language expression."
    )
```

Báo lỗi khi user instruction không chứa bất kỳ `{column}` reference nào.

## Postprocessor Default Values

### filter_postprocess (postprocessors.py:200-211)

```python
def process_outputs(answer):
    if answer is None:
        lotus.logger.info(f"\t Failed to parse {answer}: defaulting to {default}")
        return default

    if "True" in answer:
        return True
    elif "False" in answer:
        return False
    else:
        lotus.logger.info(f"\t Failed to parse {answer}: defaulting to {default}")
        return default
```

- Khi LLM output không chứa "True" hoặc "False", dùng `default` parameter (default `True` - sem_filter.py:28)
- Log info message nhưng không raise exception
- Đây là "soft failure" - operator vẫn tiếp tục hoạt động

### extract_postprocess (postprocessors.py:170-177)

```python
try:
    output = json.loads(llm_answer)
except json.JSONDecodeError:
    lotus.logger.info(f"\t Failed to parse: {llm_answer}")
    output = {}
```

Khi LLM trả về JSON không hợp lệ, return empty dict `{}`.

### parse_ans_binary cho sem_topk (sem_topk.py:127-129)

```python
except Exception:
    lotus.logger.info(f"Could not parse {answer}")
    return True, cot_explanation
```

Khi không parse được "Document 1" hay "Document 2", default chọn Document 1.

## Cascade Error Handling

### learn_filter_cascade_thresholds (sem_filter.py:195-222)

```python
try:
    large_outputs = sem_filter(...).outputs
    best_combination, _ = learn_cascade_thresholds(...)
    return best_combination
except Exception as e:
    lotus.logger.error(f"Error while learning filter cascade thresholds: {e}")
    raise e
```

Re-raise exception sau khi log.

### learn_join_cascade_threshold (sem_join.py:576-601)

```python
try:
    ...
    return pos_threshold, neg_threshold, len(sample_indices)
except Exception as e:
    lotus.logger.error(f"Error while learning filter cascade thresholds: {e}")
    lotus.logger.error("Default to full join.")
    return 1.0, 0.0, len(sample_indices)
```

Khác với filter: không re-raise, thay vào đó fallback về full join (pos=1.0, neg=0.0 nghĩa là mỗi row đều là "low confidence" và được gửi đến oracle LM).

## Cost calculation error (pricing.py:26-33)

```python
except litellm.exceptions.NotFoundError as e:
    logger.debug(f"Model pricing not found in LiteLLM: {e}")
    return None
except Exception as e:
    logger.debug(f"Unexpected error calculating completion cost: {e}")
    return None
```

Không bao giờ fail với exception từ pricing - trả về `None` và log debug. Warning được emit ở caller (lm.py:474).
