# A2 - Caching

## Hai cấp độ caching trong LOTUS

LOTUS có 2 lớp cache độc lập:
1. **LM-level cache**: Cache từng LLM response theo hash(model + messages + kwargs)
2. **Operator-level cache**: Cache toàn bộ kết quả của operator theo hash(DataFrame + arguments)

Cả hai chỉ hoạt động khi `lotus.settings.enable_cache = True` (default `False`, settings.py:17).

---

## Level 1: LM-level Cache

### Hash function (lm.py:407-410)

```python
def _hash_messages(self, messages: list[dict[str, str]], kwargs: dict[str, Any]) -> str:
    to_hash = str(self.model) + str(messages) + str(kwargs)
    return hashlib.sha256(to_hash.encode()).hexdigest()
```

Hash bao gồm: model name + full messages + all kwargs -> SHA256.

### Cache check flow trong __call__ (lm.py:136-158)

```python
if lotus.settings.enable_cache:
    hashed_messages = [self._hash_messages(msg, all_kwargs) for msg in messages]
    cached_responses_raw = [self.cache.get(hash) for hash in hashed_messages]
    # Filter None và non-ModelResponse
    ...
```

1. Hash mỗi message
2. Kiểm tra cache cho từng hash
3. Chỉ chấp nhận `ModelResponse` instances (lm.py:145-146)
4. Tách thành cached và uncached lists

### Cache insert (lm.py:392-405)

```python
def _cache_response(self, response: ModelResponse, hash: str) -> None:
    if isinstance(response, OpenAIError):
        raise response
    self.cache.insert(hash, response)
```

- Không cache error responses
- Cache sau khi nhận response thành công

### Stats tracking (lm.py:160, 174-177)

```python
self.stats.cache_hits += len(messages) - len(uncached_data)

# Update virtual stats for cached responses
if lotus.settings.enable_cache:
    for resp in cached_responses:
        if resp is not None and isinstance(resp, ModelResponse):
            self._update_stats(resp, is_cached=True)
```

- `cache_hits` đếm số responses lấy từ cache
- Virtual usage vẫn được cập nhật cho cached responses (để theo dõi tổng usage "thật")
- Physical usage chỉ cập nhật cho uncached responses (lm.py:481)

---

## Level 2: Operator-level Cache

### @operator_cache decorator (cache.py:33)

```python
def operator_cache(func: Callable) -> Callable:
    @wraps(func)
    def wrapper(self, *args, **kwargs):
        model = lotus.settings.lm
        use_operator_cache = lotus.settings.enable_cache

        if use_operator_cache and model.cache:
            # Serialize self._obj (DataFrame) + args + kwargs
            serialize_self = serialize(self._obj)
            serialized_kwargs = {key: serialize(value) for key, value in kwargs.items()}
            serialized_args = [serialize(arg) for arg in args]
            cache_key = hashlib.sha256(
                json.dumps(
                    {"self": serialize_self, "args": serialized_args, "kwargs": serialized_kwargs},
                    sort_keys=True
                ).encode()
            ).hexdigest()
```

### Serialization logic (cache.py:43-67)

`serialize()` function xử lý nhiều kiểu dữ liệu:
- `None`, `str`, `int`, `float`, `bool`: giữ nguyên
- `pd.DataFrame`: `df.to_json(orient="split")`
- `BaseModel`: `model_dump()` rồi serialize tiếp
- `list`/`tuple`: serialize từng element
- `dict`: serialize từng value
- Object có `__dict__`: serialize attributes (bỏ qua `_` prefix)
- Fallback: `str(value)` với warning

### Virtual usage tracking (cache.py:91-94)

```python
virtual_usage_before = copy.deepcopy(lotus.settings.lm.stats.virtual_usage)
result = func(self, *args, **kwargs)
virtual_usage = lotus.settings.lm.stats.virtual_usage - virtual_usage_before
model.cache.insert(virtual_usage_cache_key, virtual_usage)
```

Khi operator cache hit, virtual usage được restore (cache.py:84-86):
```python
cached_virtual_usage = model.cache.get(virtual_usage_cache_key)
if cached_virtual_usage is not None:
    model.stats.virtual_usage += cached_virtual_usage
```

Điều này đảm bảo `virtual_usage` phản ánh đúng tổng usage ngay cả khi operator được cache.

---

## Cache Implementations

### InMemoryCache (cache.py:247)

```python
class InMemoryCache(Cache):
    def __init__(self, max_size: int):
        super().__init__(max_size)
        self.cache: OrderedDict[str, Any] = OrderedDict()
```

- Sử dụng `OrderedDict` (cache.py:250)
- LRU eviction: khi vượt `max_size`, xóa entry cũ nhất (cache.py:262-263):
  ```python
  if len(self.cache) > self.max_size:
      self.cache.popitem(last=False)
  ```
- Default cho `CacheFactory.create_default_cache(max_size=1024)` (cache.py:146-147)

### SQLiteCache (cache.py:168)

```python
class SQLiteCache(Cache):
    def __init__(self, max_size: int, cache_dir=os.path.expanduser("~/.lotus/cache")):
        self.db_path = os.path.join(cache_dir, "lotus_cache.db")
```

- Persist trên disk tại `~/.lotus/cache/lotus_cache.db`
- Sử dụng `pickle.dumps/loads` để serialize values (cache.py:200, 213)
- Thread-safe qua `ThreadLocalConnection` (cache.py:150-165): mỗi thread có connection riêng
- LRU eviction dựa trên `last_accessed` timestamp (cache.py:224-238)
- Schema: `CREATE TABLE cache (key TEXT PRIMARY KEY, value BLOB, last_accessed INTEGER)` (cache.py:183-188)

### CacheFactory (cache.py:132)

```python
class CacheFactory:
    @staticmethod
    def create_cache(config: CacheConfig) -> Cache:
        if config.cache_type == CacheType.IN_MEMORY:
            return InMemoryCache(max_size=config.max_size)
        elif config.cache_type == CacheType.SQLITE:
            ...
            return SQLiteCache(max_size=config.max_size, cache_dir=cache_dir)
```

### CacheType enum (cache.py:103):
```python
class CacheType(Enum):
    IN_MEMORY = "in_memory"
    SQLITE = "sqlite"
```

---

## Lưu ý quan trọng

1. **Cache default OFF**: `enable_cache = False` (settings.py:17). User phải bật `lotus.settings.configure(enable_cache=True)`.

2. **LM luôn có cache instance**: `self.cache = cache or CacheFactory.create_default_cache()` (lm.py:121). Cache instance tồn tại ngay cả khi `enable_cache=False`, nhưng không được sử dụng.

3. **llm_as_judge disable cache**: `lotus.settings.enable_cache = False` trước khi chạy, `= True` sau khi chạy (llm_as_judge.py:82, 104). Đây là side effect nguy hiểm với concurrent code.

4. **operator_cache sử dụng LM cache instance**: Operator cache key và values được lưu trong `model.cache` (cache.py:79, 95), cùng instance với LM-level cache. Điều này có nghĩa LM responses và operator results chia sẻ cùng cache space và max_size.
