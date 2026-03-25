# Prompt Template: sem_sim_join

## Khong co LLM Prompt

`sem_sim_join` la purely embedding-based similarity join operator. No KHONG su dung LLM va KHONG co prompt template.

## Flow

```
df1.sem_sim_join(df2, left_on="article", right_on="category", K=1)
    |
    |-- Load query embeddings tu left index (sem_sim_join.py:109-119)
    |     |-- Neu left_on co index: lay vectors tu index
    |     |-- Neu khong: dung raw text
    |
    |-- Load right index (sem_sim_join.py:122-128)
    |
    |-- rm.convert_query_to_query_vector(queries)  (rm.py:53)
    |     |-- Convert left data thanh query vectors
    |
    |-- vs(query_vectors, K, ids=right_ids)        (faiss_vs.py:43)
    |     |-- Search K nearest neighbors trong right index
    |     |-- Loc theo right DataFrame indices
    |
    |-- Post-filter: bo entries voi res_id == -1     (sem_sim_join.py:142-145)
    |
    |-- Join results thanh DataFrame                  (sem_sim_join.py:147-166)
```

## Output Columns

- Tat ca columns tu left DataFrame
- Tat ca columns tu right DataFrame
- `_scores{score_suffix}`: Similarity scores (sem_sim_join.py:151)
- `_left_id`, `_right_id`: Join IDs (dropped by default, kept khi `keep_index=True`) (sem_sim_join.py:163-164)

## So sanh voi sem_join

| Aspect | sem_sim_join | sem_join |
|---|---|---|
| Model | RM (embedding) | LM (language) |
| Prompt | None | filter_formatter (via sem_filter) |
| Matching | Vector similarity score | LLM True/False judgment |
| Output | Top-K per left row | All matching pairs |
| Cost | Low (embedding) | Very high (O(n*m) LLM calls) |
| Quality | Approximate | Exact |
| Requires index | Yes (both sides) | No |

## Su dung trong Join Cascade

`sem_sim_join` duoc su dung nhu proxy model trong `sem_join` cascade (sem_join.py:336-366):

```python
def run_sem_sim_join(l1, l2, col1_label, col2_label):
    l2_df = l2_df.sem_index(col2_label, f"{col2_label}_index")
    K = len(l2)
    out = l1_df.sem_sim_join(l2_df, left_on=col1_label, right_on=col2_label, K=K, keep_index=True)
    out["_scores"] = calibrate_sem_sim_join(out["_scores"].tolist())
    return out
```

Scores tu sim_join duoc calibrate va su dung de quyet dinh pairs nao can gui den LLM oracle.
