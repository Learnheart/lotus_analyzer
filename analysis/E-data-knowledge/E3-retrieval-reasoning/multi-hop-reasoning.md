# E3 - Multi-hop Reasoning

## Tổng quan

LOTUS **KHÔNG** có explicit multi-hop reasoning support. Mỗi operator hoạt động độc lập, không có context passing giữa các steps. Multi-hop chỉ implicit qua operator chaining.

---

## 1. Không có Explicit Multi-hop

Không tìm thấy trong codebase:
- Multi-hop retrieval pipeline
- Context accumulation across steps
- Iterative retrieval-reasoning loops
- Graph-based reasoning chains

---

## 2. Implicit Multi-hop via Chaining

User có thể chain operators để tạo multi-step pipeline:

```python
result = (
    df
    .sem_search("content", "machine learning papers", K=50)     # Hop 1: retrieve
    .sem_filter("the {content} discusses neural networks")       # Hop 2: filter
    .sem_map("summarize key findings from {content}")            # Hop 3: reason
    .sem_topk("the {_output} is most impactful", K=5)           # Hop 4: rank
)
```

---

## 3. Mỗi Step Độc lập

Điểm quan trọng: mỗi operator xử lý data **independently**:

- `sem_search` chỉ thấy query và embeddings
- `sem_filter` chỉ thấy current row data và claim
- `sem_map` chỉ thấy current row data và instruction
- `sem_topk` chỉ thấy hai documents đang so sánh

**Không có**:
- Previous reasoning results passed to next step
- Accumulated context from prior hops
- Cross-step memory/state
- Chain-of-thought across operators

---

## 4. Error Propagation

Vì mỗi step xử lý output của step trước, errors propagate downstream:

```
Step 1: sem_search returns 50 docs (may miss relevant ones)
  ↓
Step 2: sem_filter keeps 20 docs (may incorrectly filter some)
  ↓
Step 3: sem_map processes 20 docs (based on potentially incomplete set)
  ↓
Step 4: sem_topk ranks 20 docs (ranking affected by missing docs)
```

- Rows bị filter sai ở step 2 → permanently lost cho step 3, 4
- sem_search miss ở step 1 → tất cả downstream steps bị ảnh hưởng
- Không có feedback loop để correct errors

---

## 5. Workaround: Manual Context Passing

User có thể manually pass context bằng sem_map enrichment:

```python
# Step 1: Add context
df = df.sem_map("Based on {content}, what related topics should we search for?", suffix="_related")

# Step 2: Use enriched data
df = df.sem_filter("the {content} and {_related} suggest relevance to deep learning")
```

Nhưng đây là manual, không automatic multi-hop.

---

## 6. Kết luận

Multi-hop reasoning trong LOTUS:
- **Implicit only**: qua operator chaining
- **Independent steps**: không có context sharing
- **Error propagation**: upstream errors affect downstream
- **No iteration**: không có retrieval-reasoning loops
- **Manual workaround**: sem_map enrichment cho context passing
