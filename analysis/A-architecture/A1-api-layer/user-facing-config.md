# A1 - User-Facing Configuration

## Settings Class

Dinh nghia tai `settings.py:8`:

```python
class Settings:
    # Models
    lm: lotus.models.LM | None = None
    rm: lotus.models.RM | None = None  # supposed to only generate embeddings
    helper_lm: lotus.models.LM | None = None
    reranker: lotus.models.Reranker | None = None
    vs: lotus.vector_store.VS | None = None

    # Cache settings
    enable_cache: bool = False

    # Serialization setting
    serialization_format: SerializationFormat = SerializationFormat.DEFAULT

    # Parallel groupby settings
    parallel_groupby_max_threads: int = 8
```

### Cac field chinh:

| Field | Type | Default | Muc dich | File:Line |
|---|---|---|---|---|
| `lm` | `LM \| None` | `None` | Language model chinh cho tat ca LLM operators | settings.py:10 |
| `rm` | `RM \| None` | `None` | Retrieval/embedding model cho vector search | settings.py:11 |
| `helper_lm` | `LM \| None` | `None` | LM phu cho cascade (nho hon, nhanh hon) | settings.py:12 |
| `reranker` | `Reranker \| None` | `None` | Cross-encoder reranker cho `sem_search` | settings.py:13 |
| `vs` | `VS \| None` | `None` | Vector store (e.g. FaissVS) | settings.py:14 |
| `enable_cache` | `bool` | `False` | Bat/tat cache cho LM responses va operator results | settings.py:17 |
| `serialization_format` | `SerializationFormat` | `DEFAULT` | Dinh dang serialize DataFrame rows | settings.py:20 |
| `parallel_groupby_max_threads` | `int` | `8` | So threads cho parallel group_by operations | settings.py:23 |

### configure() method (settings.py:25):
```python
def configure(self, **kwargs):
    for key, value in kwargs.items():
        if not hasattr(self, key):
            raise ValueError(f"Invalid setting: {key}")
        setattr(self, key, value)
```

Validate chi co key hop le, raise `ValueError` neu key khong ton tai.

## LM Constructor Parameters

Dinh nghia tai `lm.py:67`:

```python
def __init__(
    self,
    model: str = "gpt-4o-mini",
    temperature: float = 0.0,
    max_ctx_len: int = 128000,
    max_tokens: int = 512,
    max_batch_size: int = 64,
    rate_limit: int | None = None,
    tpm_limit: int | None = None,
    tokenizer: Tokenizer | None = None,
    cache: Any = None,
    physical_usage_limit: UsageLimit = UsageLimit(),
    virtual_usage_limit: UsageLimit = UsageLimit(),
    **kwargs: dict[str, Any],
):
```

| Parameter | Type | Default | Muc dich |
|---|---|---|---|
| `model` | `str` | `"gpt-4o-mini"` | Model string cho litellm (e.g. "gpt-4o", "claude-3-opus", "ollama/llama3") |
| `temperature` | `float` | `0.0` | Sampling temperature (deterministic by default) |
| `max_ctx_len` | `int` | `128000` | Max context length in tokens |
| `max_tokens` | `int` | `512` | Max tokens to generate per request |
| `max_batch_size` | `int` | `64` | Max concurrent requests in batch_completion |
| `rate_limit` | `int \| None` | `None` | Max requests per minute (RPM) |
| `tpm_limit` | `int \| None` | `None` | Max tokens per minute (TPM) |
| `tokenizer` | `Tokenizer \| None` | `None` | Custom HuggingFace tokenizer |
| `cache` | `Any` | `None` | Custom cache instance (default: InMemoryCache(1024)) |
| `physical_usage_limit` | `UsageLimit` | `UsageLimit()` | Gioi han usage thuc te (co cache) |
| `virtual_usage_limit` | `UsageLimit` | `UsageLimit()` | Gioi han usage ao (khong cache) |
| `**kwargs` | `dict` | `{}` | Extra kwargs truyen thang cho litellm API |

### UsageLimit dataclass (types.py:216):
```python
@dataclass
class UsageLimit:
    prompt_tokens_limit: float = float("inf")
    completion_tokens_limit: float = float("inf")
    total_tokens_limit: float = float("inf")
    total_cost_limit: float = float("inf")
```

## Cach user configure day du

```python
import lotus
from lotus.models import LM, SentenceTransformersRM
from lotus.vector_store import FaissVS
from lotus.types import SerializationFormat

# Basic setup
lotus.settings.configure(
    lm=LM(model="gpt-4o-mini"),
    rm=SentenceTransformersRM(model="intfloat/e5-base-v2"),
    vs=FaissVS(),
)

# Advanced setup with all options
lotus.settings.configure(
    lm=LM(
        model="gpt-4o-mini",
        temperature=0.0,
        max_ctx_len=128000,
        max_tokens=512,
        max_batch_size=64,
        rate_limit=100,            # 100 RPM
        tpm_limit=1000000,         # 1M TPM
    ),
    helper_lm=LM(model="gpt-4o-mini"),   # cho cascade operations
    rm=SentenceTransformersRM(model="intfloat/e5-base-v2"),
    vs=FaissVS(),
    enable_cache=True,
    serialization_format=SerializationFormat.JSON,
    parallel_groupby_max_threads=16,
)
```

## Luu y

1. **Thread safety**: Comment tai `settings.py:5` ghi ro "NOTE: Settings class is not thread-safe". Cac operator nhu `sem_agg` va `sem_topk` su dung `ThreadPoolExecutor` cho group_by (sem_agg.py:396-399, sem_topk.py:770-773), co the gay race condition khi doc/ghi settings.

2. **Singleton pattern**: `settings = Settings()` tai `settings.py:35` la module-level instance. Tat ca code import `lotus.settings` deu tham chieu cung 1 object.

3. **Cache default**: Khi `cache=None` trong LM constructor, se tao `InMemoryCache(max_size=1024)` qua `CacheFactory.create_default_cache()` (lm.py:121, cache.py:146-147). Nhung cache chi hoat dong khi `lotus.settings.enable_cache = True` (lm.py:136).

4. **rate_limit cap max_batch_size**: Neu `rate_limit` duoc set, `max_batch_size` duoc cap lai bang `min(rate_limit, max_batch_size)` (lm.py:108).
