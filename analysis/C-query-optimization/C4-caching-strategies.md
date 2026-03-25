# C4 - Caching Strategies trong LOTUS

## Tổng quan

LOTUS có **hai tầng cache**: LM response cache (per-call) và operator cache (per-operator-invocation).
Cả hai đều được điều khiển bởi `lotus.settings.enable_cache` (default `False`) — `settings.py:17`.

---

## 1. LM Response Cache

### Vị trí và cách hoạt động
Trong `LM.__call__` tại `lm.py:123-190`.

### Cache Key — `lm.py:407-410`
```python
def _hash_messages(self, messages, kwargs):
    to_hash = str(self.model) + str(messages) + str(kwargs)
    return hashlib.sha256(to_hash.encode()).hexdigest()
```
Key = sha256 của: `model_name + messages (string) + kwargs (string)`.

**Lưu ý:** Đây là string concatenation, không phải structured hashing. Hai messages giống nhau nhưng khác thứ tự kwargs sẽ có key khác.

### Luồng xử lý khi cache bật — `lm.py:136-190`

1. **Hash tất cả messages** — `lm.py:138`
2. **Check cache** cho mỗi message — `lm.py:139`
3. **Tách cached và uncached** — `lm.py:154-158`
4. **Track cache hits** — `lm.py:160`:
   ```python
   self.stats.cache_hits += len(messages) - len(uncached_data)
   ```
5. **Xử lý uncached** qua batch_completion — `lm.py:163-165`
6. **Lưu responses mới vào cache** — `lm.py:168-171`:
   ```python
   for resp, (_, hash) in zip(uncached_responses, uncached_data):
       self._update_stats(resp, is_cached=False)
       if lotus.settings.enable_cache:
           self._cache_response(resp, hash)
   ```
7. **Update virtual stats cho cached responses** — `lm.py:174-177`
8. **Merge responses** giữ đúng thứ tự — `lm.py:180-184`

### Cache Response method — `lm.py:392-405`
```python
def _cache_response(self, response, hash):
    if isinstance(response, OpenAIError):
        raise response  # Không cache errors
    self.cache.insert(hash, response)
```

---

## 2. Cache Implementations

### 2.1 InMemoryCache — `cache.py:247-268`

```python
class InMemoryCache(Cache):
    def __init__(self, max_size):
        self.cache: OrderedDict[str, Any] = OrderedDict()

    def get(self, key):
        return self.cache.get(key)  # O(1) lookup

    def insert(self, key, value):
        self.cache[key] = value
        if len(self.cache) > self.max_size:
            self.cache.popitem(last=False)  # LRU eviction
```

**Đặc điểm:**
- Dùng `OrderedDict` từ Python standard library — `cache.py:250`
- LRU eviction khi vượt `max_size` — `cache.py:262-263`
- **Lưu ý:** `get()` không move item lên cuối OrderedDict, nên đây không phải LRU thực sự. Items được evict theo thứ tự insert, không phải access.
- O(1) cho cả get và insert
- Mất dữ liệu khi process kết thúc

### 2.2 SQLiteCache — `cache.py:168-244`

```python
class SQLiteCache(Cache):
    def __init__(self, max_size, cache_dir="~/.lotus/cache"):
        self.db_path = os.path.join(cache_dir, "lotus_cache.db")
        self._local = threading.local()  # Thread-safe connections
```

**Đặc điểm:**
- Persistent trên disk tại `~/.lotus/cache/lotus_cache.db` — `cache.py:171`
- Thread-safe qua `threading.local()` — `cache.py:173`
- `ThreadLocalConnection` wrapper tự động close connection — `cache.py:150-165`
- Dùng `pickle` để serialize values — `cache.py:200`, `cache.py:213`
- `last_accessed` timestamp cho LRU — `cache.py:201-207`
- Size enforcement qua SQL DELETE — `cache.py:224-238`

**Schema:**
```sql
-- cache.py:183-188
CREATE TABLE IF NOT EXISTS cache (
    key TEXT PRIMARY KEY,
    value BLOB,
    last_accessed INTEGER
)
```

### 2.3 CacheFactory — `cache.py:132-147`

```python
class CacheFactory:
    @staticmethod
    def create_cache(config: CacheConfig) -> Cache:
        if config.cache_type == CacheType.IN_MEMORY:
            return InMemoryCache(max_size=config.max_size)
        elif config.cache_type == CacheType.SQLITE:
            return SQLiteCache(max_size=config.max_size, cache_dir=...)

    @staticmethod
    def create_default_cache(max_size=1024) -> Cache:
        return CacheFactory.create_cache(CacheConfig(CacheType.IN_MEMORY, max_size))
```

**Default:** `InMemoryCache(max_size=1024)` — `cache.py:146-147`.
Được tạo trong `LM.__init__` tại `lm.py:121`:
```python
self.cache = cache or CacheFactory.create_default_cache()
```

---

## 3. Operator-Level Cache (@operator_cache)

### Vị trí
Decorator `operator_cache` tại `cache.py:33-100`.

### Cách hoạt động

```python
# cache.py:33-100
def operator_cache(func):
    def wrapper(self, *args, **kwargs):
        model = lotus.settings.lm
        use_operator_cache = lotus.settings.enable_cache

        if use_operator_cache and model.cache:
            # Serialize self._obj (DataFrame) + args + kwargs
            serialize_self = serialize(self._obj)        # cache.py:69
            serialized_kwargs = {key: serialize(value) for key, value in kwargs.items()}  # cache.py:70
            serialized_args = [serialize(arg) for arg in args]  # cache.py:71

            # Create cache key
            cache_key = hashlib.sha256(
                json.dumps(
                    {"self": serialize_self, "args": serialized_args, "kwargs": serialized_kwargs},
                    sort_keys=True
                ).encode()
            ).hexdigest()  # cache.py:72-76

            # Check cache
            cached_result = model.cache.get(cache_key)  # cache.py:79
            if cached_result is not None:
                model.stats.operator_cache_hits += 1  # cache.py:82
                # Restore virtual usage stats
                cached_virtual_usage = model.cache.get(virtual_usage_cache_key)  # cache.py:84
                if cached_virtual_usage is not None:
                    model.stats.virtual_usage += cached_virtual_usage  # cache.py:86
                return cached_result  # cache.py:88

            # Execute and cache result
            virtual_usage_before = copy.deepcopy(lotus.settings.lm.stats.virtual_usage)  # cache.py:91
            result = func(self, *args, **kwargs)  # cache.py:92
            virtual_usage = lotus.settings.lm.stats.virtual_usage - virtual_usage_before  # cache.py:93
            model.cache.insert(virtual_usage_cache_key, virtual_usage)  # cache.py:94
            model.cache.insert(cache_key, result)  # cache.py:95
            return result

        return func(self, *args, **kwargs)  # cache.py:98
```

### Serialization logic — `cache.py:43-67`

Hàm `serialize` nội bộ xử lý nhiều kiểu dữ liệu:
- `None`, `str`, `int`, `float`, `bool` → giữ nguyên — `cache.py:48-49`
- `pd.DataFrame` → `df.to_json(orient="split")` — `cache.py:51`
- `BaseModel` (Pydantic) → `model_dump()` — `cache.py:53`
- `list`, `tuple` → recursive serialize — `cache.py:56-57`
- `dict` → recursive serialize — `cache.py:58-59`
- Fallback → `str(value)` với warning — `cache.py:66-67`

### Virtual usage tracking — `cache.py:77-95`

**Vấn đề:** Khi operator result được cache, LLM không được gọi. Nhưng `virtual_usage` (tổng usage nếu không có cache) vẫn cần được track để báo cáo chính xác.

**Giải pháp:**
1. Trước khi chạy operator: ghi nhận `virtual_usage_before` — `cache.py:91`
2. Sau khi chạy: tính `delta = current - before` — `cache.py:93`
3. Lưu delta vào cache với key `cache_key + "_usage"` — `cache.py:94`
4. Khi cache hit: cộng delta vào `model.stats.virtual_usage` — `cache.py:86`

### Các operators sử dụng @operator_cache
- `SemFilterDataframe.__call__` — `sem_filter.py:333`
- `SemMapDataframe.__call__` — `sem_map.py:214`
- `SemTopKDataframe.__call__` — `sem_topk.py:734`
- `SemAggDataframe.__call__` — `sem_agg.py:353`
- `SemJoinDataframe.__call__` — `sem_join.py:669`
- `SemExtractDataFrame.__call__` — `sem_extract.py:201`
- `SemSearchDataframe.__call__` — `sem_search.py:91`
- `SemDedupByDataframe.__call__` — `sem_dedup.py:32`
- `SemSimJoinDataframe.__call__` — `sem_sim_join.py:84`
- `SemClusterByDataframe.__call__` — `sem_cluster_by.py:57`
- `SemPartitionByDataframe.__call__` — `sem_partition_by.py:60`

---

## 4. Usage Statistics

### LMStats — `types.py:20-66`

```python
@dataclass
class LMStats:
    @dataclass
    class TotalUsage:
        prompt_tokens: int = 0
        completion_tokens: int = 0
        total_tokens: int = 0
        total_cost: float = 0.0
        cached_prompt_tokens: int = 0      # API-level cache hits
        cache_creation_tokens: int = 0     # API-level cache writes

    virtual_usage: TotalUsage   # Usage nếu không có cache
    physical_usage: TotalUsage  # Usage thực tế (có cache)
    cache_hits: int = 0              # LM response cache hits
    operator_cache_hits: int = 0     # Operator cache hits
```

**Hai loại usage:**
- `virtual_usage`: Luôn được update, kể cả khi cache hit — `lm.py:477`
- `physical_usage`: Chỉ update khi không cache hit — `lm.py:481-483`

---

## 5. Hạn chế và Lưu ý

1. **Cache key collision risk thấp nhưng không zero:** String concatenation có thể tạo collision khi model name chứa ký tự giống messages.

2. **InMemoryCache không thread-safe:** OrderedDict không có lock. Khi dùng ThreadPoolExecutor (group_by), có thể xảy ra race condition.

3. **operator_cache cache DataFrame:** Với DataFrame lớn, bộ nhớ có thể bị đầy nhanh. max_size=1024 có thể là quá lớn cho operator cache.

4. **SQLiteCache dùng pickle:** Pickle không an toàn với untrusted data. Không nên chia sẻ cache file giữa các người dùng.

5. **enable_cache default False:** User phải chủ động bật cache. Đây là safe default nhưng có thể làm user bỏ lỡ optimization.
