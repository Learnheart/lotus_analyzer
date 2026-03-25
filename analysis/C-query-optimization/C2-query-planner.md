# C2 - Query Planner trong LOTUS (Han che)

## Tong quan

LOTUS co **query planning RAT HAN CHE**. Chi co duy nhat `join_optimizer` tai `sem_join.py:417` thuc su la mot optimizer.
Khong co global query plan nao cho chuoi cac operators duoc chain voi nhau.

---

## 1. join_optimizer — Optimizer duy nhat

### Vi tri
`sem_join.py:417-527`

### Chuc nang
So sanh hai strategies de tim plan re nhat, dam bao recall va precision target.

### Hai strategies duoc so sanh

#### Strategy 1: Search-Filter (SF)
1. Chay `run_sem_sim_join(l1, l2)` — `sem_join.py:463`
   - Dung embedding similarity lam proxy scores
   - `sem_sim_join` tra ve DataFrame voi `_scores` column
2. `calibrate_sem_sim_join` — `cascade_utils.py:147-149` — clip scores ve [0, 1]
3. `learn_join_cascade_threshold` — `sem_join.py:530-603`
   - Sample mot phan data (`importance_sampling` — `cascade_utils.py:8-30`)
   - Chay oracle LM tren sample — `sem_join.py:577-587`
   - Tim optimal (pos_threshold, neg_threshold) — `sem_join.py:589-594`
4. Phan loai:
   - `_scores >= pos_threshold` → accept (high confidence positive) — `sem_join.py:477`
   - `_scores <= neg_threshold` → reject (high confidence negative) — `sem_join.py:478`
   - Con lai → can oracle LM — `sem_join.py:479-480`

#### Strategy 2: Map-Search-Filter (MSF)
1. `map_l1_to_l2(l1, col1_label, col2_label)` — `sem_join.py:369-414`
   - Dung `sem_map` de "dich" l1 sang domain cua l2
   - Vi du: "Cho product name, liet ke 2-10 categories lien quan"
   - Chi phi: len(l1) LLM calls — `sem_join.py:504`
2. Chay `run_sem_sim_join` tren mapped l1 — `sem_join.py:486`
3. Tuong tu SF: learn thresholds, phan loai high/low confidence

### Quyet dinh chon plan

```python
# sem_join.py:518-527
if sf_cost < msf_cost:
    lotus.logger.info("Proceeding with Search-Filter")
    return sf_high_conf, sf_low_conf, sf_high_conf_neg, learning_cost
else:
    lotus.logger.info("Proceeding with Map-Search-Filter")
    return msf_high_conf, msf_low_conf, msf_high_conf_neg, learning_cost
```

- `sf_cost = len(sf_low_conf)` — so LLM calls cho Search-Filter — `sem_join.py:480`
- `msf_cost = len(msf_low_conf)` — so LLM calls cho Map-Search-Filter — `sem_join.py:503`
- Khong tinh den chi phi learning (da bo ra roi)
- Chi phi MSF co them `len(l1)` tu buoc map — `sem_join.py:504`

### Log output
```python
# sem_join.py:507-515
lotus.logger.info("Join Optimizer: plan cost analysis:")
lotus.logger.info(f"    Search-Filter: {sf_cost} LLM calls.")
lotus.logger.info(f"    Map-Search-Filter: {msf_cost} LLM calls.")
```

---

## 2. Khong co Global Query Plan

### Van de
Khi user viet:
```python
df.sem_filter("Is {text} about AI?")
  .sem_map("Summarize {text}")
  .sem_topk("Most relevant {_map}", K=5)
```

**Moi operator chay doc lap, khong biet ve operator sau:**
- `sem_filter` goi LLM cho **tat ca** rows truoc
- `sem_map` goi LLM cho tat ca rows con lai
- `sem_topk` goi LLM cho quicksort comparisons

**Nhung mot optimizer thong minh co the:**
- Push `sem_topk` vao truoc `sem_map` (vi sem_topk chi can K rows)
- Ket hop `sem_filter` + `sem_map` thanh mot LLM call
- Pre-filter bang embedding truoc khi chay bat ky LLM operator nao

### Nguyen nhan
- LOTUS dung pandas accessor pattern — moi operator la mot method call doc lap
- Khong co IR (Intermediate Representation) hay execution graph
- Khong co lazy evaluation — tat ca la eager execution
- Thiet ke uu tien don gian va de hieu hon la toi uu hieu suat

---

## 3. Cascade Threshold Learning — Planning noi bo

Du khong co global planner, moi cascade operator co "micro-planning" noi bo:

### Filter cascade planning — `sem_filter.py:395-462`
1. Chay proxy model (helper_lm hoac embedding) tren TOAN BO data
2. `importance_sampling` de chon sample representative — `cascade_utils.py:8-30`
3. Chay oracle LM tren sample — `sem_filter.py:196-208`
4. `learn_cascade_thresholds` de tim optimal thresholds — `cascade_utils.py:42-144`
5. Ap dung thresholds: high-confidence → proxy, low-confidence → oracle — `sem_filter.py:467-530`

### Join cascade planning — `sem_join.py:242-257`
1. `join_optimizer` chay CA HAI strategies va chon re hon
2. Moi strategy co `learn_join_cascade_threshold` rieng — `sem_join.py:464-476` va `sem_join.py:487-499`

### TopK cascade — `sem_topk.py:401-455`
1. Khong co threshold learning truoc
2. Cascade duoc ap dung on-the-fly trong moi partition step
3. Helper LM chay truoc, kiem tra confidence tren logprobs — `sem_topk.py:248-252`
4. Low-confidence comparisons gui den large LM — `sem_topk.py:257-272`

---

## 4. safe_mode — Uoc luong chi phi (Khong phai Optimizer)

`safe_mode` cho phep uoc luong chi phi truoc khi thuc thi, nhung **khong thay doi execution plan**.

**Ho tro:**
- `sem_filter`: `sem_filter.py:107-110` — uoc luong token count cho tat ca inputs
- `sem_join`: `sem_join.py:104-120` — uoc luong len(l1) * len(l2) calls
- `sem_topk` quicksort: `sem_topk.py:393-399` — uoc luong 2K + 2N*log(N) calls
- `sem_topk` heapsort: `sem_topk.py:597-603` — uoc luong N*log(N) + K*log(N) calls

**KHONG ho tro (TODO):**
- `sem_agg`: `sem_agg.py:152-153` — "Safe mode is not implemented yet"
- `sem_join` cascade: `sem_join.py:262-264` — "Safe mode is not implemented yet"

---

## 5. Tom tat

| Khia canh | Trang thai | Chi tiet |
|-----------|-----------|----------|
| Global query planner | KHONG CO | Eager execution, trai-sang-phai |
| Join optimizer | CO | `sem_join.py:417` — SF vs MSF |
| Filter cascade | CO | Threshold learning + routing |
| TopK cascade | CO | On-the-fly confidence routing |
| Operator reordering | KHONG CO | Khong co IR/graph |
| Cost estimation | MOT PHAN | safe_mode cho mot so operators |
| Predicate pushdown | KHONG CO | Khong co co che nao |
| Common subexpression elimination | KHONG CO | Tuy operator_cache co the cover mot phan |
