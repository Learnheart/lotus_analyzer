# A1 - Method Registry: Tất cả sem_* Methods

## Bảng tổng hợp tất cả các method

| Method | Full Signature | File:Line | Return Type | Description |
|---|---|---|---|---|
| `sem_filter` | `__call__(self, user_instruction: str, return_raw_outputs: bool = False, return_explanations: bool = False, return_all: bool = False, default: bool = True, suffix: str = "_filter", examples: pd.DataFrame \| None = None, helper_examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, cascade_args: CascadeArgs \| None = None, return_stats: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Filtering", additional_cot_instructions: str = "") -> pd.DataFrame \| tuple[pd.DataFrame, dict[str, Any]]` | `sem_ops/sem_filter.py:334` | `pd.DataFrame` hoặc `tuple[pd.DataFrame, dict]` | Lọc DataFrame theo điều kiện ngôn ngữ tự nhiên. Trả về rows thỏa mãn điều kiện, hoặc tất cả rows với label True/False nếu `return_all=True`. |
| `sem_map` | `__call__(self, user_instruction: str, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_explanations: bool = False, return_raw_outputs: bool = False, suffix: str = "_map", examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Mapping", **model_kwargs: Any) -> pd.DataFrame` | `sem_ops/sem_map.py:215` | `pd.DataFrame` | Áp dụng phép biến đổi ngôn ngữ trên mỗi row. Kết quả được thêm vào cột mới với suffix. |
| `sem_join` | `__call__(self, other: pd.DataFrame \| pd.Series, join_instruction: str, return_explanations: bool = False, how: str = "inner", suffix: str = "_join", examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, default: bool = True, cascade_args: CascadeArgs \| None = None, return_stats: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Join comparisons") -> pd.DataFrame` | `sem_ops/sem_join.py:670` | `pd.DataFrame` | Join 2 DataFrame bằng semantic matching. Mỗi cặp (row1, row2) được đánh giá bởi LLM qua `sem_filter` nội bộ. Chỉ hỗ trợ inner join. |
| `sem_agg` | `__call__(self, user_instruction: str, all_cols: bool = False, suffix: str = "_output", group_by: list[str] \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Aggregating", long_context_strategy: LongContextStrategy \| None = LongContextStrategy.CHUNK) -> pd.DataFrame` | `sem_ops/sem_agg.py:354` | `pd.DataFrame` | Tổng hợp nhiều rows thành 1 câu trả lời. Sử dụng hierarchical aggregation (cây), hỗ trợ group_by. |
| `sem_topk` | `__call__(self, user_instruction: str, K: int, method: str = "quick", strategy: ReasoningStrategy \| None = None, group_by: list[str] \| None = None, cascade_threshold: float \| None = None, return_stats: bool = False, safe_mode: bool = False, return_explanations: bool = False) -> pd.DataFrame \| tuple[pd.DataFrame, dict[str, Any]]` | `sem_ops/sem_topk.py:735` | `pd.DataFrame` hoặc `tuple` | Sắp xếp và trả về K rows phù hợp nhất. Hỗ trợ 4 thuật toán: quick, heap, naive, quick-sem. |
| `sem_extract` | `__call__(self, input_cols: list[str], output_cols: dict[str, str \| None], extract_quotes: bool = False, postprocessor: Callable = extract_postprocess, return_raw_outputs: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Extracting", return_explanations: bool = False, strategy: ReasoningStrategy \| None = None) -> pd.DataFrame` | `sem_ops/sem_extract.py:202` | `pd.DataFrame` | Trích xuất thông tin có cấu trúc (JSON) từ mỗi row. Mỗi output_col trở thành 1 cột mới. |
| `sem_search` | `__call__(self, col_name: str, query: str, K: int \| None = None, n_rerank: int \| None = None, return_scores: bool = False, suffix: str = "_sim_score") -> pd.DataFrame` | `sem_ops/sem_search.py:92` | `pd.DataFrame` | Tìm kiếm vector (embedding-based). Yêu cầu `sem_index` trước. Hỗ trợ reranking. |
| `sem_sim_join` | `__call__(self, other: pd.DataFrame, left_on: str, right_on: str, K: int, lsuffix: str = "", rsuffix: str = "", score_suffix: str = "", keep_index: bool = False) -> pd.DataFrame` | `sem_ops/sem_sim_join.py:85` | `pd.DataFrame` | Join 2 DataFrame bằng embedding similarity. Mỗi row của left được match với K rows gần nhất của right. |
| `sem_dedup` | `__call__(self, col_name: str, threshold: float) -> pd.DataFrame` | `sem_ops/sem_dedup.py:33` | `pd.DataFrame` | Loại bỏ duplicate bằng embedding similarity. Sử dụng connected components trên đồ thị similarity. |
| `sem_index` | `__call__(self, col_name: str, index_dir: str) -> pd.DataFrame` | `sem_ops/sem_index.py:62` | `pd.DataFrame` | Tạo vector index cho 1 column. Gọi `RM.__call__` để embedding, `VS.index()` để lưu. |
| `load_sem_index` | `__call__(self, col_name: str, index_dir: str) -> pd.DataFrame` | `sem_ops/load_sem_index.py:49` | `pd.DataFrame` | Load vector index từ disk. Chỉ set `attrs["index_dirs"]`, không load index vào memory. |
| `sem_partition_by` | `__call__(self, partition_fn: Callable[[pd.DataFrame], list[int]]) -> pd.DataFrame` | `sem_ops/sem_partition_by.py:61` | `pd.DataFrame` | Gán partition ID cho mỗi row. Được dùng trước `sem_agg` để nhóm dữ liệu. |
| `sem_cluster_by` | `__call__(self, col_name: str, ncentroids: int, return_scores: bool = False, return_centroids: bool = False, niter: int = 20, verbose: bool = False) -> pd.DataFrame \| tuple[pd.DataFrame, np.ndarray]` | `sem_ops/sem_cluster_by.py:58` | `pd.DataFrame` | Clustering bằng faiss k-means trên embeddings. Thêm cột `cluster_id`. |
| `llm_as_judge` | `__call__(self, judge_instruction: str, response_format: BaseModel \| None = None, n_trials: int = 1, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_raw_outputs: bool = False, return_explanations: bool = False, suffix: str = "_judge", examples: pd.DataFrame \| None = None, cot_reasoning: list[str] \| None = None, strategy: ReasoningStrategy \| None = None, extra_cols_to_include: list[str] \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Evaluating", **model_kwargs: Any) -> pd.DataFrame` | `evals/llm_as_judge.py:188` | `pd.DataFrame` | Đánh giá mỗi row bằng LLM. Nội bộ gọi `sem_map` với n_trials. Disable cache khi chạy. |
| `pairwise_judge` | `__call__(self, col1: str, col2: str, judge_instruction: str, response_format: BaseModel \| None = None, n_trials: int = 1, permute_cols: bool = False, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_raw_outputs: bool = False, return_explanations: bool = False, suffix: str = "_judge", examples: pd.DataFrame \| None = None, cot_reasoning: list[str] \| None = None, strategy: ReasoningStrategy \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Evaluating", **model_kwargs: Any) -> pd.DataFrame` | `evals/pairwise_judge.py:70` | `pd.DataFrame` | So sánh cặp-đôi 2 columns. Nội bộ gọi `llm_as_judge`. Hỗ trợ permute_cols để giảm position bias. |

## Phân loại theo dependency

### Chỉ cần LM (lotus.settings.lm)
- `sem_filter`, `sem_map`, `sem_join`, `sem_agg`, `sem_topk`, `sem_extract`, `llm_as_judge`, `pairwise_judge`

### Chỉ cần RM + VS (lotus.settings.rm, lotus.settings.vs)
- `sem_search`, `sem_sim_join`, `sem_dedup`, `sem_index`, `sem_cluster_by`

### Không cần model nào
- `load_sem_index`, `sem_partition_by`

### Cần cả LM và RM/VS (khi dùng cascade)
- `sem_filter` với `cascade_args` và `proxy_model=ProxyModel.EMBEDDING_MODEL` (sem_filter.py:435-441)
- `sem_join` với `cascade_args` (gọi `sem_sim_join` nội bộ) (sem_join.py:746-773)
- `sem_topk` với `method="quick-sem"` (gọi `sem_index` + `sem_search`) (sem_topk.py:782-788)

## Shared infrastructure

Tất cả LLM operators chia sẻ:
- `@operator_cache` decorator (cache.py:33)
- `lotus.nl_expression.parse_cols()` (nl_expression.py:4) để extract column names từ `{col}` syntax
- `lotus.nl_expression.nle2str()` (nl_expression.py:17) để format instruction string
- `task_instructions.df2multimodal_info()` (task_instructions.py:364) để convert DataFrame rows thành multimodal dicts
