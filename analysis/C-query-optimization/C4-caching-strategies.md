# C4 - Caching Strategies trong LOTUS

## Tong quan

LOTUS co **hai tang cache**: LM response cache (per-call) va operator cache (per-operator-invocation).
Ca hai deu duoc dieu khien boi `lotus.settings.enable_cache` (default `False`) — `settings.py:17`.

---

## 1. LM Response Cache

### Vi tri va cach hoat dong
Trong `LM.__call__` tai `lm.py:123-190`.

### Cache Key — `lm.py:407-410`
```python
def _hash_messages(self, messages, kwargs):
    to_hash = str(self.model) + str(messages) + str(kwargs)
    return hashlib.sha256(to_hash.encode()).hexdigest()
```
Key = sha256 cua: `model_name + messages (string) + kwargs (string)`.

**Luu y:** Day la string concatenation, khong phai structured hashing. Hai messages giong nhau nhung khac thu tu kwargs se co key khac.

### Luong xu ly khi cache bat — `lm.py:136-190`

1. **Hash tat ca messages** — `lm.py:138`
2. **Check cache** cho moi message — `lm.py:139`
3. **Tach cached va uncached** — `lm.py:154-158`
4. **Track cache hits** — `lm.py:160`:
   ```python
   self.stats.cache_hits += len(messages) - len(uncached_data)
   ```
5. **Xu ly uncached** qua batch_completion — `lm.py:163-165`
6. **Luu responses moi vao cache** — `lm.py:168-171`:
   ```python
   for resp, (_, hash) in zip(uncached_responses, uncached_data):
       self._update_stats(resp, is_cached=False)
       if lotus.settings.enable_cache:
           self._cache_response(resp, hash)
   ```
7. **Update virtual stats cho cached responses** — `lm.py:174-177`
8. **Merge responses** giu dung thu tu — `lm.py:180-184`

### Cache Response method — `lm.py:392-405`
```python
def _cache_response(self, response, hash):
    if isinstance(response, OpenAIError):
        raise response  # Khong cache errors
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

**Dac diem:**
- Dung `OrderedDict` tu Python standard library — `cache.py:250`
- LRU eviction khi vuot `max_size` — `cache.py:262-263`
- **Luu y:** `get()` khong move item len cuoi OrderedDict, nen day khong phai LRU thuc su. Items duoc evict theo thu tu insert, khong phai access.
- O(1) cho ca get va insert
- Mat du lieu khi process ket thuc

### 2.2 SQLiteCache — `cache.py:168-244`

```python
class SQLiteCache(Cache):
    def __init__(self, max_size, cache_dir="~/.lotus/cache"):
        self.db_path = os.path.join(cache_dir, "lotus_cache.db")
        self._local = threading.local()  # Thread-safe connections
```

**Dac diem:**
- Persistent tren disk tai `~/.lotus/cache/lotus_cache.db` — `cache.py:171`
- Thread-safe qua `threading.local()` — `cache.py:173`
- `ThreadLocalConnection` wrapper tu dong close connection — `cache.py:150-165`
- Dung `pickle` de serialize values — `cache.py:200`, `cache.py:213`
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
Duoc tao trong `LM.__init__` tai `lm.py:121`:
```python
self.cache = cache or CacheFactory.create_default_cache()
```

---

## 3. Operator-Level Cache (@operator_cache)

### Vi tri
Decorator `operator_cache` tai `cache.py:33-100`.

### Cach hoat dong

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

Ham `serialize` noi bo xu ly nhieu kieu du lieu:
- `None`, `str`, `int`, `float`, `bool` → giu nguyen — `cache.py:48-49`
- `pd.DataFrame` → `df.to_json(orient="split")` — `cache.py:51`
- `BaseModel` (Pydantic) → `model_dump()` — `cache.py:53`
- `list`, `tuple` → recursive serialize — `cache.py:56-57`
- `dict` → recursive serialize — `cache.py:58-59`
- Fallback → `str(value)` voi warning — `cache.py:66-67`

### Virtual usage tracking — `cache.py:77-95`

**Van de:** Khi operator result duoc cache, LLM khong duoc goi. Nhung `virtual_usage` (tong usage neu khong co cache) van can duoc track de bao cao chinh xac.

**Giai phap:**
1. Truoc khi chay operator: ghi nhan `virtual_usage_before` — `cache.py:91`
2. Sau khi chay: tinh `delta = current - before` — `cache.py:93`
3. Luu delta vao cache voi key `cache_key + "_usage"` — `cache.py:94`
4. Khi cache hit: cong delta vao `model.stats.virtual_usage` — `cache.py:86`

### Cac operators su dung @operator_cache
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

    virtual_usage: TotalUsage   # Usage neu khong co cache
    physical_usage: TotalUsage  # Usage thuc te (co cache)
    cache_hits: int = 0              # LM response cache hits
    operator_cache_hits: int = 0     # Operator cache hits
```

**Hai loai usage:**
- `virtual_usage`: Luon duoc update, ke ca khi cache hit — `lm.py:477`
- `physical_usage`: Chi update khi khong cache hit — `lm.py:481-483`

---

## 5. Han che va Luu y

1. **Cache key collision risk thap nhung khong zero:** String concatenation co the tao collision khi model name chua ky tu giong messages.

2. **InMemoryCache khong thread-safe:** OrderedDict khong co lock. Khi dung ThreadPoolExecutor (group_by), co the xay ra race condition.

3. **operator_cache cache DataFrame:** Voi DataFrame lon, bo nho co the bi day nhanh. max_size=1024 co the la qua lon cho operator cache.

4. **SQLiteCache dung pickle:** Pickle khong an toan voi untrusted data. Khong nen chia se cache file giua cac nguoi dung.

5. **enable_cache default False:** User phai chu dong bat cache. Day la safe default nhung co the lam user bo lo optimization.
