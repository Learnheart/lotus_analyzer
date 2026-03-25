# Operator Cheatsheet — LOTUS

## Semantic Operators

| Operator | Description | SQL Equivalent | Input | Output | Langex Example | Use Case |
|----------|-------------|----------------|-------|--------|----------------|----------|
| `sem_filter` | Lọc rows theo điều kiện ngôn ngữ tự nhiên | `WHERE <semantic_predicate>` | DataFrame + instruction | DataFrame (filtered) | `df.sem_filter("The {review} is positive")` | Lọc sản phẩm được đánh giá tốt |
| `sem_map` | Ánh xạ mỗi row thành output mới qua LLM | `SELECT LLM(col) AS new_col` | DataFrame + instruction | DataFrame + new column | `df.sem_map("Summarize {text} in one sentence")` | Trích xuất sentiment, tóm tắt |
| `sem_extract` | Trích xuất structured attributes từ text | `SELECT extract(col, schema)` | DataFrame + input_cols + output_cols | DataFrame + extracted columns | `df.sem_extract(["text"], {"sentiment": "pos/neg"})` | Trích xuất entities, attributes |
| `sem_agg` | Tổng hợp nhiều rows thành một output | `SELECT LLM_AGG(col)` | DataFrame + instruction | DataFrame (1 row) | `df.sem_agg("Summarize all {entries}")` | Tóm tắt nhiều documents |
| `sem_topk` | Xếp hạng và chọn K rows tốt nhất | `ORDER BY <semantic_rank> LIMIT K` | DataFrame + instruction + K | DataFrame (K rows) | `df.sem_topk("Most relevant {title}", K=3)` | Tìm top reviews, best matches |
| `sem_join` | Join hai DataFrames theo điều kiện semantic | `JOIN ON <semantic_predicate>` | 2 DataFrames + instruction | DataFrame (joined) | `df1.sem_join(df2, "{product} matches {category}")` | Mapping products to categories |
| `sem_search` | Tìm kiếm semantic qua embeddings | `WHERE vec_sim(col, query) > threshold` | DataFrame + col + query + K | DataFrame (K rows) | `df.sem_search("title", "AI tutorial", K=5)` | Tìm tài liệu liên quan |
| `sem_sim_join` | Join theo embedding similarity | `JOIN ON vec_sim(a, b) TOP K` | 2 DataFrames + left/right cols + K | DataFrame (joined + scores) | `df1.sem_sim_join(df2, "article", "category", K=2)` | Tìm K categories gần nhất cho mỗi article |
| `sem_dedup` | Loại bỏ duplicates theo semantic similarity | `DISTINCT ON semantic_sim` | DataFrame + col + threshold | DataFrame (deduped) | `df.sem_dedup("title", threshold=0.9)` | Loại bỏ tin trùng lặp |
| `sem_cluster_by` | Phân cụm theo embedding similarity | `GROUP BY semantic_cluster` | DataFrame + col + ncentroids | DataFrame + cluster_id | `df.sem_cluster_by("title", 3)` | Phân nhóm tài liệu |
| `sem_partition_by` | Phân vùng DataFrame theo function | `PARTITION BY function` | DataFrame + partition_fn | DataFrame + partition_id | `df.sem_partition_by(cluster("col", 2))` | Phân vùng trước aggregation |
| `sem_index` | Tạo vector index cho column | `CREATE INDEX` | DataFrame + col + index_dir | DataFrame (with index metadata) | `df.sem_index("title", "title_idx")` | Chuẩn bị cho search/join |
| `load_sem_index` | Load vector index từ disk | `LOAD INDEX` | DataFrame + col + index_dir | DataFrame (with index metadata) | `df.load_sem_index("title", "title_idx")` | Tải lại index đã tạo |

## Evaluation Operators

| Operator | Description | Input | Output | Example |
|----------|-------------|-------|--------|---------|
| `llm_as_judge` | Đánh giá output bằng LLM | DataFrame + instruction | DataFrame + scores | Đánh giá chất lượng answers |
| `pairwise_judge` | So sánh cặp output bằng LLM | DataFrame + pairs | DataFrame + preferences | So sánh hai model outputs |

## Web Operators

| Operator | Description | Input | Output | Example |
|----------|-------------|-------|--------|---------|
| `web_search` | Tìm kiếm web | Query | WebSearchCorpus | Tìm thông tin trên internet |
| `web_extract` | Trích xuất nội dung từ web | URLs | DataFrame | Lấy nội dung trang web |

---

## Comparison Matrix: Tính năng nâng cao

| Operator | Cascade Support | GroupBy Support | CoT/ZS-COT | safe_mode | Reranker |
|----------|----------------|-----------------|-------------|-----------|----------|
| sem_filter | Có (helper_lm / embedding) | Không | Có | Có | Không |
| sem_map | Không | Không | Có | Có | Không |
| sem_extract | Không | Không | Có | Có | Không |
| sem_agg | Không | Có (ThreadPoolExecutor) | Không | TODO | Không |
| sem_topk | Có (helper_lm) | Có (ThreadPoolExecutor) | Có (ZS_COT) | Có | Không |
| sem_join | Có (embedding + optimizer) | Không | Có | Một phần | Không |
| sem_search | Không | Không | Không | Không | Có |
| sem_sim_join | Không | Không | Không | Không | Không |
| sem_dedup | Không | Không | Không | Không | Không |
| sem_cluster_by | Không | Không | Không | Không | Không |

---

## Sorting Algorithms trong sem_topk

| Method | Complexity | Cascade | Embedding Pivot | File:Line |
|--------|-----------|---------|-----------------|-----------|
| `quick` | O(n log n) avg | Có | Không | `sem_topk.py:347-488` |
| `quick-sem` | O(n log n) avg | Có | Có | `sem_topk.py:782-800` |
| `heap` | O(n + K log n) | Không | Không | `sem_topk.py:560-621` |
| `naive` | O(n^2) | Không | Không | `sem_topk.py:276-344` |
