# E5 - Consistency Checks

## Tổng quan

LOTUS **KHÔNG** có self-consistency hay majority voting trong main operators. Một số mechanisms tồn tại trong evaluation framework nhưng không trong core operations.

---

## 1. Không có Self-consistency trong Main Operators

Các operators chính (sem_filter, sem_map, sem_extract, sem_topk, sem_agg) đều:
- Chạy MỘT lần per input
- Nhận MỘT output per input
- Không verify output
- Không retry khi output bất thường

---

## 2. llm_as_judge: n_trials

**Location**: `evals/llm_as_judge.py:16-114`

```python
def llm_as_judge(docs, model, judge_instruction,
                 n_trials=1, ...):              # :22
```

Khi `n_trials > 1`:
```python
with ThreadPoolExecutor(...) as executor:
    sem_map_outputs = list(
        executor.map(
            lambda _: sem_map(docs, model, judge_instruction, ...),
            range(n_trials),                    # :101
        )
    )
```

- Chạy evaluation `n_trials` lần song song
- Cache disabled: `lotus.settings.enable_cache = False` (`llm_as_judge.py:82`)
- Nhưng **KHÔNG** aggregate results - trả về list of individual trial outputs

User phải tự aggregate:
```python
# Manual majority voting
results = df.llm_as_judge("judge {output}", n_trials=5)
# results has _judge_0, _judge_1, ..., _judge_4 columns
# User must compute majority vote manually
```

---

## 3. pairwise_judge: permute_cols

**Location**: `evals/pairwise_judge.py:100-140`

```python
if permute_cols:
    if n_trials % 2:
        raise ValueError("Number of trials should be even")    # :101-102
    for c1, c2 in [(col1, col2), (col2, col1)]:                # :105-107
        output = self._obj.pairwise_judge(
            col1=c1, col2=c2, ...
            n_trials=n_trials // 2,                              # :114
        )
```

Position bias mitigation:
- Chạy n_trials/2 với (A, B) order
- Chạy n_trials/2 với (B, A) order
- Giảm bias do LLM prefer first/last document

Nhưng vẫn **KHÔNG** auto-aggregate - user nhận tất cả trial results.

---

## 4. Không có Cross-validation

Không tìm thấy:
- Cross-validation giữa operators (verify sem_filter results với sem_map)
- Ensemble methods (multiple models voting)
- Output validation against constraints
- Semantic consistency checks (same input → same output)

---

## 5. Temperature 0 as Implicit Consistency

Temperature 0.0 default (`lm.py:69`) → deterministic output cho same input:
- Same prompt → same response (ignoring API non-determinism)
- Đây là form yếu nhất của consistency - chỉ reproducibility, không verification

Nhưng lưu ý: different API calls có thể vẫn return slightly different results dù temperature=0.

---

## 6. Caching as Consistency

Cache mechanism (`lm.py:136-151`):
```python
hashed_messages = [self._hash_messages(msg, all_kwargs) for msg in messages]
cached_responses_raw = [self.cache.get(hash) for hash in hashed_messages]
```

Nếu cùng input → trả về cached response → guaranteed consistent.
Nhưng đây là optimization, không phải consistency check.

---

## 7. Kết luận

Consistency trong LOTUS:
- **Không có** self-consistency/majority voting trong core operators
- **n_trials** trong llm_as_judge nhưng không auto-aggregate
- **permute_cols** cho position bias nhưng user phải tự interpret
- **Temperature 0** cho reproducibility
- **Caching** cho identical inputs
- **Gap**: Thiếu built-in consistency verification
