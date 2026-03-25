# Operator Cheatsheet — LOTUS

## Semantic Operators

| Operator | Description | SQL Equivalent | Input | Output | Langex Example | Use Case |
|----------|-------------|----------------|-------|--------|----------------|----------|
| `sem_filter` | Loc rows theo dieu kien ngon ngu tu nhien | `WHERE <semantic_predicate>` | DataFrame + instruction | DataFrame (filtered) | `df.sem_filter("The {review} is positive")` | Loc san pham duoc danh gia tot |
| `sem_map` | Anh xa moi row thanh output moi qua LLM | `SELECT LLM(col) AS new_col` | DataFrame + instruction | DataFrame + new column | `df.sem_map("Summarize {text} in one sentence")` | Trich xuat sentiment, tom tat |
| `sem_extract` | Trich xuat structured attributes tu text | `SELECT extract(col, schema)` | DataFrame + input_cols + output_cols | DataFrame + extracted columns | `df.sem_extract(["text"], {"sentiment": "pos/neg"})` | Trich xuat entities, attributes |
| `sem_agg` | Tong hop nhieu rows thanh mot output | `SELECT LLM_AGG(col)` | DataFrame + instruction | DataFrame (1 row) | `df.sem_agg("Summarize all {entries}")` | Tom tat nhieu documents |
| `sem_topk` | Xep hang va chon K rows tot nhat | `ORDER BY <semantic_rank> LIMIT K` | DataFrame + instruction + K | DataFrame (K rows) | `df.sem_topk("Most relevant {title}", K=3)` | Tim top reviews, best matches |
| `sem_join` | Join hai DataFrames theo dieu kien semantic | `JOIN ON <semantic_predicate>` | 2 DataFrames + instruction | DataFrame (joined) | `df1.sem_join(df2, "{product} matches {category}")` | Mapping products to categories |
| `sem_search` | Tim kiem semantic qua embeddings | `WHERE vec_sim(col, query) > threshold` | DataFrame + col + query + K | DataFrame (K rows) | `df.sem_search("title", "AI tutorial", K=5)` | Tim tai lieu lien quan |
| `sem_sim_join` | Join theo embedding similarity | `JOIN ON vec_sim(a, b) TOP K` | 2 DataFrames + left/right cols + K | DataFrame (joined + scores) | `df1.sem_sim_join(df2, "article", "category", K=2)` | Tim K categories gan nhat cho moi article |
| `sem_dedup` | Loai bo duplicates theo semantic similarity | `DISTINCT ON semantic_sim` | DataFrame + col + threshold | DataFrame (deduped) | `df.sem_dedup("title", threshold=0.9)` | Loai bo tin trung lap |
| `sem_cluster_by` | Phan cum theo embedding similarity | `GROUP BY semantic_cluster` | DataFrame + col + ncentroids | DataFrame + cluster_id | `df.sem_cluster_by("title", 3)` | Phan nhom tai lieu |
| `sem_partition_by` | Phan vung DataFrame theo function | `PARTITION BY function` | DataFrame + partition_fn | DataFrame + partition_id | `df.sem_partition_by(cluster("col", 2))` | Phan vung truoc aggregation |
| `sem_index` | Tao vector index cho column | `CREATE INDEX` | DataFrame + col + index_dir | DataFrame (with index metadata) | `df.sem_index("title", "title_idx")` | Chuan bi cho search/join |
| `load_sem_index` | Load vector index tu disk | `LOAD INDEX` | DataFrame + col + index_dir | DataFrame (with index metadata) | `df.load_sem_index("title", "title_idx")` | Tai lai index da tao |

## Evaluation Operators

| Operator | Description | Input | Output | Example |
|----------|-------------|-------|--------|---------|
| `llm_as_judge` | Danh gia output bang LLM | DataFrame + instruction | DataFrame + scores | Danh gia chat luong answers |
| `pairwise_judge` | So sanh cap output bang LLM | DataFrame + pairs | DataFrame + preferences | So sanh hai model outputs |

## Web Operators

| Operator | Description | Input | Output | Example |
|----------|-------------|-------|--------|---------|
| `web_search` | Tim kiem web | Query | WebSearchCorpus | Tim thong tin tren internet |
| `web_extract` | Trich xuat noi dung tu web | URLs | DataFrame | Lay noi dung trang web |

---

## Comparison Matrix: Tinh nang nang cao

| Operator | Cascade Support | GroupBy Support | CoT/ZS-COT | safe_mode | Reranker |
|----------|----------------|-----------------|-------------|-----------|----------|
| sem_filter | Co (helper_lm / embedding) | Khong | Co | Co | Khong |
| sem_map | Khong | Khong | Co | Co | Khong |
| sem_extract | Khong | Khong | Co | Co | Khong |
| sem_agg | Khong | Co (ThreadPoolExecutor) | Khong | TODO | Khong |
| sem_topk | Co (helper_lm) | Co (ThreadPoolExecutor) | Co (ZS_COT) | Co | Khong |
| sem_join | Co (embedding + optimizer) | Khong | Co | Mot phan | Khong |
| sem_search | Khong | Khong | Khong | Khong | Co |
| sem_sim_join | Khong | Khong | Khong | Khong | Khong |
| sem_dedup | Khong | Khong | Khong | Khong | Khong |
| sem_cluster_by | Khong | Khong | Khong | Khong | Khong |

---

## Sorting Algorithms trong sem_topk

| Method | Complexity | Cascade | Embedding Pivot | File:Line |
|--------|-----------|---------|-----------------|-----------|
| `quick` | O(n log n) avg | Co | Khong | `sem_topk.py:347-488` |
| `quick-sem` | O(n log n) avg | Co | Co | `sem_topk.py:782-800` |
| `heap` | O(n + K log n) | Khong | Khong | `sem_topk.py:560-621` |
| `naive` | O(n^2) | Khong | Khong | `sem_topk.py:276-344` |
