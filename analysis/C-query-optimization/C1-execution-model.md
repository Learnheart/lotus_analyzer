# C1 - Mo hinh Execution cua LOTUS

## Tong quan

LOTUS khong co query planner toan cuc. Cac operator thuc thi **eager, trai-sang-phai** theo chuoi method calls tren DataFrame.
Moi operator doc lap goi LM hoac retrieval model, xu ly ket qua, va tra ve DataFrame moi.

---

## 1. Eager Execution — Khong co Query Plan

**Khong co IR (Intermediate Representation):**
- Moi operator la mot pandas DataFrame accessor — dang ky qua `@pd.api.extensions.register_dataframe_accessor`
- Vi du: `sem_filter.py:225`, `sem_map.py:121`, `sem_topk.py:624`, `sem_agg.py:226`, `sem_join.py:606`
- Khi user viet `df.sem_filter(...).sem_map(...)`, Python thuc thi tuan tu: `sem_filter` chay xong, tra ve DataFrame moi, roi `sem_map` chay tren ket qua do

**He qua:**
- Khong co co hoi reorder operators (vi du: push filter xuong truoc join)
- Khong co cost-based optimization toan cuc
- Moi operator phai hoat dong doc lap, khong biet gi ve operator tiep theo

---

## 2. Luong xu ly trong moi Operator

Moi operator semantic thuc hien cac buoc tuong tu:

### Buoc 1: Parse langex va validate columns
```
col_li = lotus.nl_expression.parse_cols(user_instruction)
# nl_expression.py:4-8 — regex match {column_name}
```
Kiem tra columns ton tai trong DataFrame — vi du `sem_filter.py:363-365`.

### Buoc 2: Chuyen DataFrame thanh multimodal data
```
multimodal_data = task_instructions.df2multimodal_info(self._obj, col_li)
# Moi row -> dict chua text + images
```
Vi du tai `sem_filter.py:367`, `sem_map.py:241`, `sem_topk.py:790`.

### Buoc 3: Format user instruction
```
formatted_usr_instr = lotus.nl_expression.nle2str(user_instruction, col_li)
# nl_expression.py:17-21 — thay {col} bang "Col"
```

### Buoc 4: Tao prompts va goi LM
- Moi operator co formatter rieng (filter_formatter, map_formatter, extract_formatter)
- Gui batch prompts den `model(inputs)` — goi `LM.__call__` tai `lm.py:123`
- LM xu ly batching noi bo — `lm.py:250-252`

### Buoc 5: Post-process va tra ve DataFrame
- `filter_postprocess` tai `postprocessors.py:182-218`: parse True/False
- `map_postprocess` tai `postprocessors.py:123-146`: raw output hoac CoT extraction
- `extract_postprocess` tai `postprocessors.py:149-179`: JSON parsing
- Ket qua duoc gan vao DataFrame moi va tra ve

---

## 3. GroupBy Parallelism qua ThreadPoolExecutor

**Cac operator ho tro group_by:**
- `sem_agg`: `sem_agg.py:381-399` — grouped aggregation
- `sem_topk`: `sem_topk.py:763-780` — grouped top-k

**Cach hoat dong:**
```python
# sem_agg.py:396-399
from concurrent.futures import ThreadPoolExecutor
with ThreadPoolExecutor(max_workers=lotus.settings.parallel_groupby_max_threads) as executor:
    return pd.concat(list(executor.map(SemAggDataframe.process_group, group_args)))
```

- `process_group` la static method xu ly moi group doc lap — `sem_agg.py:325-351`
- Max threads cau hinh tai `settings.py:23`: `parallel_groupby_max_threads: int = 8`
- Tuong tu cho `sem_topk.py:770-773`

**Luu y:** Day la thread-level parallelism, khong phai process-level. Do GIL cua Python, hieu qua chu yeu den tu I/O-bound work (API calls).

---

## 4. Batch Processing trong LM

**Trong `LM.__call__`** tai `lm.py:123-190`:

1. **Check cache** — tach cached vs uncached messages — `lm.py:136-158`
2. **Xu ly uncached** qua `_process_uncached_messages` — `lm.py:215-256`:
   - Khong co rate/tpm limit: `batch_completion(model, batch, max_workers=max_batch_size)` — `lm.py:250-252`
   - Co rate limit: chia sub-batches, sleep giua cac batch — `lm.py:258-303`
   - Co tpm limit: tinh token budget, gui tung sub-batch vua du — `lm.py:311-390`
3. **Merge responses** — `lm.py:412-417`
4. **Extract outputs** — `lm.py:185-190`

**max_batch_size** mac dinh 64 — `lm.py:73`. Neu co `rate_limit`, bi cap boi `min(rate_limit, max_batch_size)` — `lm.py:108`.

---

## 5. Hierarchical Aggregation Tree

`sem_agg` co execution model dac biet — khong chi goi LM mot lan.

**Tai `sem_agg.py:60-223`:**
- Docs duoc nhom lai cho den khi vua context window
- Moi nhom tao mot prompt, gui den LM
- Ket qua (summaries) tro thanh input cho level tiep theo
- Lap lai cho den khi chi con 1 summary — `sem_agg.py:164`
- Su dung template khac nhau cho leaf vs node — `sem_agg.py:12-57`

**Do phuc tap:** O(n / context_capacity) LLM calls moi level, O(log n) levels => O(n * log(n) / context_capacity) tong LLM calls.

---

## 6. So do luong xu ly tong the

```
User Code: df.sem_filter("...").sem_map("...").sem_topk("...", K=5)
            |                    |                    |
            v                    v                    v
     [sem_filter.__call__]  [sem_map.__call__]  [sem_topk.__call__]
            |                    |                    |
     1. Parse langex       1. Parse langex      1. Parse langex
     2. df -> multimodal   2. df -> multimodal  2. df -> multimodal
     3. Format prompts     3. Format prompts    3. Build pairs
     4. LM(batch)          4. LM(batch)         4. LM(comparisons)
        |                     |                    |
        v                     v                    v
     [LM.__call__]         [LM.__call__]        [LM.__call__]
     - Check cache         - Check cache        - Check cache
     - batch_completion    - batch_completion   - batch_completion
     - Update stats        - Update stats       - Update stats
        |                     |                    |
        v                     v                    v
     5. Postprocess        5. Postprocess       5. Sort indexes
     6. Return DataFrame   6. Return DataFrame  6. Return DataFrame
```

**Key insight:** Moi operator la mot "island" doc lap. Khong co data flow optimization giua cac operators.
