# E2 - Knowledge Graph Potential

## Tổng quan

LOTUS hiện tại **KHÔNG** có explicit knowledge graph construction. Tuy nhiên, một số operators tạo ra implicit graph-like structures có thể được tận dụng.

---

## 1. Không có Explicit KG Construction

Không có module nào trong LOTUS codebase:
- Tạo nodes/edges
- Lưu trữ graph structure
- Query graph relationships
- Export sang graph format (RDF, Neo4j, etc.)

---

## 2. Implicit Links qua sem_join

`sem_join` tạo implicit relationships giữa hai bảng:

```python
# sem_join.py:157-163
join_results.extend([
    (all_ids1[i], all_ids2[i], explanation)
    for i, (output, explanation) in enumerate(zip(outputs, explanations))
    if output
])
```

Output `join_results` là list of `(id1, id2, explanation)` tuples (`types.py:143`):
```python
join_results: list[tuple[int, int, str | None]]
```

Đây chính là edge list trong graph terminology:
- `id1`, `id2` = nodes
- `explanation` = edge label/description
- Relationship = user-defined qua join_instruction

---

## 3. sem_cluster_by - Grouping

`sem_cluster_by` nhóm các items tương tự nhau dựa trên embedding similarity.

Cluster function (`utils.py:14-71`):
```python
def cluster(col_name, ncentroids):
    def ret(df, niter=20, verbose=False, method="kmeans"):
        kmeans = faiss.Kmeans(d, ncentroids, niter=niter, verbose=verbose)  # utils.py:61
        kmeans.train(vec_set)                                                # utils.py:62
        scores, indices = kmeans.index.search(vec_set, 1)                   # utils.py:65
        return indices.flatten()                                             # utils.py:70
```

Cluster IDs có thể được coi là implicit community membership trong graph.

---

## 4. Potential: Chain Operators cho KG Construction

Có thể xây dựng KG bằng cách chain LOTUS operators:

### Step 1: Entity Extraction
```python
df = df.sem_extract(
    ["text"],
    {"entity1": "first entity", "entity2": "second entity", "relationship": "relationship type"}
)
```

### Step 2: Entity Resolution via sem_join
```python
entities_df.sem_join(entities_df, "the {entity:left} refers to the same thing as {entity:right}")
```

### Step 3: Graph Construction
```python
# join_results gives us edges
# entities give us nodes
# Can export to NetworkX, Neo4j, etc.
```

### Step 4: Community Detection via sem_cluster_by
```python
entities_df.sem_cluster_by("entity", ncentroids=10)
```

---

## 5. Limitations

- **No native graph storage**: Kết quả phải export ra external graph database
- **No graph traversal**: Không có multi-hop query trên graph
- **No entity resolution built-in**: sem_join có thể dùng nhưng không optimized cho entity matching
- **Cost**: Mỗi step đều cần LLM calls, expensive cho large-scale KG construction
- **No incremental updates**: Không thể thêm edges/nodes incrementally

---

## 6. Kết luận

LOTUS có potential cho KG construction qua operator chaining, nhưng đây là emergent capability, không phải designed feature. Cần significant engineering effort để biến thành production-ready KG pipeline.
