# A2 - Caching

## Hai cap do caching trong LOTUS

LOTUS co 2 lop cache doc lap:
1. **LM-level cache**: Cache tung LLM response theo hash(model + messages + kwargs)
2. **Operator-level cache**: Cache toan bo ket qua cua operator theo hash(DataFrame + arguments)

Ca hai chi hoat dong khi `lotus.settings.enable_cache = True` (default `False`, settings.py:17).

---

## Level 1: LM-level Cache

### Hash function (lm.py:407-410)

```python
def _hash_messages(self, messages: list[dict[str, str]], kwargs: dict[str, Any]) -> str:
    to_hash = str(self.model) + str(messages) + str(kwargs)
    return hashlib.sha256(to_hash.encode()).hexdigest()
```

Hash bao gom: model name + full messages + all kwargs -> SHA256.

### Cache check flow trong __call__ (lm.py:136-158)

```python
if lotus.settings.enable_cache:
    hashed_messages = [self._hash_messages(msg, all_kwargs) for msg in messages]
    cached_responses_raw = [self.cache.get(hash) for hash in hashed_messages]
    # Filter None va non-ModelResponse
    ...
```

1. Hash moi message
2. Kiem tra cache cho tung hash
3. Chi chap nhan `ModelResponse` instances (lm.py:145-146)
4. Tach thanh cached va uncached lists

### Cache insert (lm.py:392-405)

```python
def _cache_response(self, response: ModelResponse, hash: str) -> None:
    if isinstance(response, OpenAIError):
        raise response
    self.cache.insert(hash, response)
```

- Khong cache error responses
- Cache sau khi nhan response thanh cong

### Stats tracking (lm.py:160, 174-177)

```python
self.stats.cache_hits += len(messages) - len(uncached_data)

# Update virtual stats for cached responses
if lotus.settings.enable_cache:
    for resp in cached_responses:
        if resp is not None and isinstance(resp, ModelResponse):
            self._update_stats(resp, is_cached=True)
```

- `cache_hits` dem so responses lay tu cache
- Virtual usage van duoc cap nhat cho cached responses (de theo doi tong usage "that")
- Physical usage chi cap nhat cho uncached responses (lm.py:481)

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

`serialize()` function xu ly nhieu kieu du lieu:
- `None`, `str`, `int`, `float`, `bool`: giu nguyen
- `pd.DataFrame`: `df.to_json(orient="split")`
- `BaseModel`: `model_dump()` roi serialize tiep
- `list`/`tuple`: serialize tung element
- `dict`: serialize tung value
- Object co `__dict__`: serialize attributes (bo qua `_` prefix)
- Fallback: `str(value)` voi warning

### Virtual usage tracking (cache.py:91-94)

```python
virtual_usage_before = copy.deepcopy(lotus.settings.lm.stats.virtual_usage)
result = func(self, *args, **kwargs)
virtual_usage = lotus.settings.lm.stats.virtual_usage - virtual_usage_before
model.cache.insert(virtual_usage_cache_key, virtual_usage)
```

Khi operator cache hit, virtual usage duoc restore (cache.py:84-86):
```python
cached_virtual_usage = model.cache.get(virtual_usage_cache_key)
if cached_virtual_usage is not None:
    model.stats.virtual_usage += cached_virtual_usage
```

Dieu nay dam bao `virtual_usage` phan anh dung tong usage ngay ca khi operator duoc cache.

---

## Cache Implementations

### InMemoryCache (cache.py:247)

```python
class InMemoryCache(Cache):
    def __init__(self, max_size: int):
        super().__init__(max_size)
        self.cache: OrderedDict[str, Any] = OrderedDict()
```

- Su dung `OrderedDict` (cache.py:250)
- LRU eviction: khi vuot `max_size`, xoa entry cu nhat (cache.py:262-263):
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

- Persist tren disk tai `~/.lotus/cache/lotus_cache.db`
- Su dung `pickle.dumps/loads` de serialize values (cache.py:200, 213)
- Thread-safe qua `ThreadLocalConnection` (cache.py:150-165): moi thread co connection rieng
- LRU eviction dua tren `last_accessed` timestamp (cache.py:224-238)
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

## Luu y quan trong

1. **Cache default OFF**: `enable_cache = False` (settings.py:17). User phai bat `lotus.settings.configure(enable_cache=True)`.

2. **LM luon co cache instance**: `self.cache = cache or CacheFactory.create_default_cache()` (lm.py:121). Cache instance ton tai ngay ca khi `enable_cache=False`, nhung khong duoc su dung.

3. **llm_as_judge disable cache**: `lotus.settings.enable_cache = False` truoc khi chay, `= True` sau khi chay (llm_as_judge.py:82, 104). Day la side effect nguy hiem voi concurrent code.

4. **operator_cache su dung LM cache instance**: Operator cache key va values duoc luu trong `model.cache` (cache.py:79, 95), cung instance voi LM-level cache. Dieu nay co nghia LM responses va operator results chia se cung cache space va max_size.
