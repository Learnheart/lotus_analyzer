# A2 - LLM Client Abstraction

## LM Class Overview

Dinh nghia tai `models/lm.py:41`. Day la lop trung tam cua LOTUS cho tuong tac voi LLM providers.

```
LM (lm.py:41)
  |
  |-- __call__(messages) -> LMOutput        (lm.py:123)
  |-- get_completion(system, user) -> str    (lm.py:192)
  |-- count_tokens(messages) -> int          (lm.py:550)
  |-- encode_text(text) -> list[int]         (lm.py:565)
  |-- decode_tokens(tokens) -> str           (lm.py:572)
  |-- print_total_usage()                    (lm.py:579)
  |-- reset_stats()                          (lm.py:589)
  |-- reset_cache()                          (lm.py:592)
  |-- format_logprobs_for_cascade()          (lm.py:507)
  |-- format_logprobs_for_filter_cascade()   (lm.py:517)
```

## Provider Switching qua litellm

LOTUS khong truc tiep implement API call cho tung provider. Thay vao do, su dung `litellm.batch_completion` (import tai lm.py:10) lam abstraction layer.

### Cach doi provider:
Chi can thay doi model string:
- OpenAI: `LM(model="gpt-4o-mini")`, `LM(model="gpt-4o")`
- Anthropic: `LM(model="claude-3-opus")`, `LM(model="claude-3-5-sonnet")`
- Ollama (local): `LM(model="ollama/llama3")`
- DeepSeek: `LM(model="deepseek/deepseek-r1")`

### Batch completion call (lm.py:250-251):
```python
uncached_responses = batch_completion(
    self.model, batch, drop_params=True, max_workers=self.max_batch_size, **all_kwargs
)
```

- `drop_params=True`: litellm tu dong bo cac params khong duoc ho tro boi provider cu the
- `max_workers=self.max_batch_size`: so luong concurrent workers

## __call__ Method Flow (lm.py:123)

```python
def __call__(
    self,
    messages: list[list[dict[str, str]]],
    show_progress_bar: bool = True,
    progress_bar_desc: str = "Processing uncached messages",
    **kwargs: dict[str, Any],
) -> LMOutput:
```

Luong xu ly:

1. **Merge kwargs** (lm.py:130): `all_kwargs = {**self.kwargs, **kwargs}`
2. **Set logprobs** (lm.py:133-134): Neu `logprobs=True`, set `top_logprobs=10`
3. **Cache check** (lm.py:136-158):
   - Hash messages + kwargs -> `_hash_messages()` (lm.py:407)
   - Kiem tra cache cho tung message
   - Tach thanh cached va uncached
4. **Process uncached** (lm.py:163-165): `_process_uncached_messages()`
   - Neu `tpm_limit`: `_process_with_tpm_limiting()` (lm.py:311)
   - Neu `rate_limit`: `_process_with_rate_limiting()` (lm.py:258)
   - Khong co limit: `batch_completion()` truc tiep (lm.py:250)
5. **Update stats** (lm.py:168-177):
   - Physical stats cho uncached responses
   - Virtual stats cho tat ca responses
6. **Merge responses** (lm.py:180-184): Gop cached + uncached theo thu tu goc
7. **Extract outputs** (lm.py:185-190): `_get_top_choice()` -> text string

## get_completion() (lm.py:192)

Helper method tien loi cho 1 call don:

```python
def get_completion(
    self,
    system_prompt: str,
    user_prompt: str,
    show_progress_bar: bool = True,
    progress_bar_desc: str = "Processing uncached messages",
    response_format: BaseModel | None = None,
    **kwargs: dict[str, Any],
) -> str | BaseModel:
```

- Wrap system + user thanh messages format
- Goi `self()` (tuc `__call__`)
- Neu `response_format` duoc cung cap, parse output thanh Pydantic model

## DeepSeek Detection (lm.py:612)

```python
def is_deepseek(self) -> bool:
    model_name = self.get_model_name()
    return model_name.startswith("deepseek-r1")
```

Duoc su dung trong `filter_formatter` (task_instructions.py:152) va `map_formatter` (task_instructions.py:249) de them deepseek-specific CoT instructions vao prompt.

## Tokenizer Support

LM ho tro custom tokenizer (lm.py:113, lm.py:550-563):
```python
def count_tokens(self, messages: list[dict[str, str]] | str) -> int:
    custom_tokenizer: dict[str, Any] | None = None
    if self.tokenizer:
        custom_tokenizer = dict(type="huggingface_tokenizer", tokenizer=self.tokenizer)
    return token_counter(
        custom_tokenizer=custom_tokenizer,
        model=self.model,
        messages=messages,
    )
```

Su dung `litellm.utils.token_counter`, `encode`, `decode` (import tai lm.py:13).
