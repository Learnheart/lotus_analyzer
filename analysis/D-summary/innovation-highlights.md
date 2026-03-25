# Innovation Highlights — LOTUS

## Top 7 Innovations voi File:Line References

---

## 1. Pandas Accessor Pattern cho Seamless DataFrame Integration

**Vi tri:** Tat ca operator files, vi du `sem_filter.py:225`, `sem_map.py:121`, `sem_topk.py:624`

**Innovation:**
LOTUS dang ky moi semantic operator nhu mot pandas DataFrame accessor qua `@pd.api.extensions.register_dataframe_accessor`. Dieu nay cho phep:

```python
# Tich hop tu nhien vao pandas workflow
df.sem_filter("Is {text} positive?").sem_map("Summarize {text}").sem_topk("Best {_map}", K=5)
```

**Tai sao dot pha:**
- Khong can DSL rieng hay API moi — dung truc tiep tren pandas
- Chainable — ket qua cua operator nay la input cua operator tiep theo
- Zero migration cost — DataFrame hien tai van dung duoc
- Moi operator la mot class voi `__init__` nhan `pandas_obj` va `__call__` thuc thi logic — vi du `sem_filter.py:308-316`

**Langex (Natural Language Expressions):**
User tham chieu columns bang `{column_name}` — parse boi regex tai `nl_expression.py:4-8`:
```python
pattern = r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})"
```

---

## 2. Model Cascading voi Statistical Guarantees

**Vi tri:** `cascade_utils.py:42-144`, `sem_filter.py:383-530`, `sem_join.py:180-333`, `sem_topk.py:176-273`

**Innovation:**
He thong tu dong hoc cascade thresholds tu data, dam bao recall va precision targets voi **dam bao thong ke (statistical guarantees)**.

**Co so toan hoc:**
- Hoeffding-style concentration inequalities — `cascade_utils.py:52-56`
- Importance sampling voi correction factors — `cascade_utils.py:8-30`
- Corrected recall target qua UB/LB bounds — `cascade_utils.py:118-121`
- Precision guarantee voi Bonferroni correction — `cascade_utils.py:127-137`

**Tai sao dot pha:**
- Khong chi la "dung model nho truoc" — co ly thuyet dam bao do chinh xac
- User chi can set `recall_target=0.9, precision_target=0.9` va he thong tu tim thresholds
- Giam chi phi 50-90% so voi chay full oracle model

---

## 3. Join Optimizer (Search-Filter vs Map-Search-Filter)

**Vi tri:** `sem_join.py:417-527`

**Innovation:**
LOTUS co query optimizer cho semantic join — so sanh hai strategies va chon plan re nhat:

1. **Search-Filter (SF):** Embedding similarity → filter uncertain pairs voi oracle LLM
2. **Map-Search-Filter (MSF):** LLM map left table → embedding similarity → filter

**Quyet dinh:** `sem_join.py:518-527`
```python
if sf_cost < msf_cost:
    return sf_high_conf, sf_low_conf, sf_high_conf_neg, learning_cost
else:
    return msf_high_conf, msf_low_conf, msf_high_conf_neg, learning_cost
```

**Tai sao dot pha:**
- Day la cost-based optimizer cho LLM queries — rat it framework lam duoc
- Tu dong chon strategy tot nhat dua tren data characteristics
- Map step (MSF) co the giam search space dang ke khi domains khac nhau nhieu

---

## 4. Hierarchical Aggregation Tree cho Unbounded Data

**Vi tri:** `sem_agg.py:60-223`

**Innovation:**
Khi data lon hon context window, sem_agg xay dung cay aggregation da cap:
- Level 0: Nhom documents thanh batches vua context window, moi batch → summary
- Level 1+: Nhom summaries thanh batches, moi batch → higher-level summary
- Lap lai cho den khi chi con 1 final summary — `sem_agg.py:164`

**Template khac nhau cho moi level:**
- Leaf template — `sem_agg.py:12-31`: xu ly raw documents
- Node template — `sem_agg.py:34-57`: xu ly summaries tu cac sources khac nhau

**Partition-aware:** Documents cung partition_id duoc aggregation voi nhau — `sem_agg.py:179-184`.

**Tai sao dot pha:**
- Xu ly du lieu KHONG GIOI HAN — khong bi gioi han boi context window
- Dam bao thong tin tu tat ca documents deu duoc xem xet
- Hieu qua: O(n * log(n) / context_capacity) LLM calls

---

## 5. Multimodal Support qua ImageDtype Extension

**Vi tri:** `dtype_extensions/image.py:12-35` (ImageDtype), `dtype_extensions/image.py:37-305` (ImageArray)

**Innovation:**
LOTUS tao custom pandas ExtensionDtype cho images, cho phep luu tru va xu ly images truc tiep trong DataFrame.

```python
class ImageDtype(ExtensionDtype):
    name = "image"
    type = Image.Image
    na_value = None
```

**Dac diem:**
- `ImageArray` ho tro lazy loading va caching — `image.py:117-132`
- Comparison logic cho images — `image.py:307-327`
- Tu dong convert giua Image, base64, va file paths
- Tich hop voi multimodal LLM prompts qua `task_instructions.df2multimodal_info`

**Tai sao dot pha:**
- Khong chi xu ly text — DataFrame co the chua ca images
- Images duoc xu ly nhu "first-class citizens" trong moi operator
- Mot API thong nhat cho ca text va vision tasks

---

## 6. Langex — Natural Language Expressions lam Declarative Query Interface

**Vi tri:** `nl_expression.py:1-29`

**Innovation:**
Langex la cach LOTUS cho phep user viet queries bang ngon ngu tu nhien voi column references:

```python
# Langex syntax
df.sem_filter("The {review} expresses a positive sentiment about {product}")
df.sem_map("Translate {text} to French")
df.sem_topk("The {title} is most relevant to machine learning", K=5)
```

**Cach hoat dong:**
1. `parse_cols` — `nl_expression.py:4-8`: extract column names tu `{...}`
2. `nle2str` — `nl_expression.py:17-21`: thay `{col}` bang `Col.capitalize()`
3. Double braces `{{...}}` duoc ignore (escaped) — `nl_expression.py:6`

**Tai sao dot pha:**
- Declarative — user noi "cai gi" chu khong phai "lam the nao"
- Natural language lam bridge giua structured data (columns) va unstructured reasoning (LLM)
- De hoc, de su dung — khong can hoc DSL phuc tap

---

## 7. LLM-Based Pairwise Comparison Algorithms (Quicksort, Heapsort)

**Vi tri:** `sem_topk.py:347-488` (quicksort), `sem_topk.py:560-621` (heapsort), `sem_topk.py:276-344` (naive sort)

**Innovation:**
LOTUS implement cac thuat toan sorting co dien nhung thay comparator function bang LLM call:

**Quicksort** — `sem_topk.py:347-488`:
- Partition dung LLM pairwise comparisons — `sem_topk.py:407-463`
- Chi sort top-K (khong can sort toan bo) — `sem_topk.py:465-483`
- Ho tro cascade threshold — `sem_topk.py:437-455`
- Ho tro embedding-based pivot selection — `sem_topk.py:411-417`

**Heapsort** — `sem_topk.py:560-621`:
- `HeapDoc` class override `__lt__` de goi LLM — `sem_topk.py:526-557`
- `heapq.nsmallest(K, heap)` — chi lay K items — `sem_topk.py:613`
- O(n + K log n) LLM calls

**Naive sort** — `sem_topk.py:276-344`:
- All-pairs comparison + voting — `sem_topk.py:310-339`
- O(n^2) LLM calls — chi dung cho dataset nho

**Tai sao dot pha:**
- Ap dung computer science algorithms len LLM reasoning
- Quicksort voi top-K pruning giam tu O(n^2) xuong O(n log n) avg
- Cascade + embedding pivot tiep tuc giam chi phi
- Ket qua la ranking dua tren LLM judgment — khong phai embedding similarity
