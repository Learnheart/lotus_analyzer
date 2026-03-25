# A1 - Method Registry: Tat ca sem_* Methods

## Bang tong hop tat ca cac method

| Method | Full Signature | File:Line | Return Type | Description |
|---|---|---|---|---|
| `sem_filter` | `__call__(self, user_instruction: str, return_raw_outputs: bool = False, return_explanations: bool = False, return_all: bool = False, default: bool = True, suffix: str = "_filter", examples: pd.DataFrame \| None = None, helper_examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, cascade_args: CascadeArgs \| None = None, return_stats: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Filtering", additional_cot_instructions: str = "") -> pd.DataFrame \| tuple[pd.DataFrame, dict[str, Any]]` | `sem_ops/sem_filter.py:334` | `pd.DataFrame` hoac `tuple[pd.DataFrame, dict]` | Loc DataFrame theo dieu kien ngon ngu tu nhien. Tra ve rows thoa man dieu kien, hoac tat ca rows voi label True/False neu `return_all=True`. |
| `sem_map` | `__call__(self, user_instruction: str, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_explanations: bool = False, return_raw_outputs: bool = False, suffix: str = "_map", examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Mapping", **model_kwargs: Any) -> pd.DataFrame` | `sem_ops/sem_map.py:215` | `pd.DataFrame` | Ap dung phep bien doi ngon ngu tren moi row. Ket qua duoc them vao cot moi voi suffix. |
| `sem_join` | `__call__(self, other: pd.DataFrame \| pd.Series, join_instruction: str, return_explanations: bool = False, how: str = "inner", suffix: str = "_join", examples: pd.DataFrame \| None = None, strategy: ReasoningStrategy \| None = None, default: bool = True, cascade_args: CascadeArgs \| None = None, return_stats: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Join comparisons") -> pd.DataFrame` | `sem_ops/sem_join.py:670` | `pd.DataFrame` | Join 2 DataFrame bang semantic matching. Moi cap (row1, row2) duoc danh gia boi LLM qua `sem_filter` noi bo. Chi ho tro inner join. |
| `sem_agg` | `__call__(self, user_instruction: str, all_cols: bool = False, suffix: str = "_output", group_by: list[str] \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Aggregating", long_context_strategy: LongContextStrategy \| None = LongContextStrategy.CHUNK) -> pd.DataFrame` | `sem_ops/sem_agg.py:354` | `pd.DataFrame` | Tong hop nhieu rows thanh 1 cau tra loi. Su dung hierarchical aggregation (cay), ho tro group_by. |
| `sem_topk` | `__call__(self, user_instruction: str, K: int, method: str = "quick", strategy: ReasoningStrategy \| None = None, group_by: list[str] \| None = None, cascade_threshold: float \| None = None, return_stats: bool = False, safe_mode: bool = False, return_explanations: bool = False) -> pd.DataFrame \| tuple[pd.DataFrame, dict[str, Any]]` | `sem_ops/sem_topk.py:735` | `pd.DataFrame` hoac `tuple` | Sap xep va tra ve K rows phu hop nhat. Ho tro 4 thuat toan: quick, heap, naive, quick-sem. |
| `sem_extract` | `__call__(self, input_cols: list[str], output_cols: dict[str, str \| None], extract_quotes: bool = False, postprocessor: Callable = extract_postprocess, return_raw_outputs: bool = False, safe_mode: bool = False, progress_bar_desc: str = "Extracting", return_explanations: bool = False, strategy: ReasoningStrategy \| None = None) -> pd.DataFrame` | `sem_ops/sem_extract.py:202` | `pd.DataFrame` | Trich xuat thong tin co cau truc (JSON) tu moi row. Moi output_col tro thanh 1 cot moi. |
| `sem_search` | `__call__(self, col_name: str, query: str, K: int \| None = None, n_rerank: int \| None = None, return_scores: bool = False, suffix: str = "_sim_score") -> pd.DataFrame` | `sem_ops/sem_search.py:92` | `pd.DataFrame` | Tim kiem vector (embedding-based). Yeu cau `sem_index` truoc. Ho tro reranking. |
| `sem_sim_join` | `__call__(self, other: pd.DataFrame, left_on: str, right_on: str, K: int, lsuffix: str = "", rsuffix: str = "", score_suffix: str = "", keep_index: bool = False) -> pd.DataFrame` | `sem_ops/sem_sim_join.py:85` | `pd.DataFrame` | Join 2 DataFrame bang embedding similarity. Moi row cua left duoc match voi K rows gan nhat cua right. |
| `sem_dedup` | `__call__(self, col_name: str, threshold: float) -> pd.DataFrame` | `sem_ops/sem_dedup.py:33` | `pd.DataFrame` | Loai bo duplicate bang embedding similarity. Su dung connected components tren do thi similarity. |
| `sem_index` | `__call__(self, col_name: str, index_dir: str) -> pd.DataFrame` | `sem_ops/sem_index.py:62` | `pd.DataFrame` | Tao vector index cho 1 column. Goi `RM.__call__` de embedding, `VS.index()` de luu. |
| `load_sem_index` | `__call__(self, col_name: str, index_dir: str) -> pd.DataFrame` | `sem_ops/load_sem_index.py:49` | `pd.DataFrame` | Load vector index tu disk. Chi set `attrs["index_dirs"]`, khong load index vao memory. |
| `sem_partition_by` | `__call__(self, partition_fn: Callable[[pd.DataFrame], list[int]]) -> pd.DataFrame` | `sem_ops/sem_partition_by.py:61` | `pd.DataFrame` | Gan partition ID cho moi row. Duoc dung truoc `sem_agg` de nhom du lieu. |
| `sem_cluster_by` | `__call__(self, col_name: str, ncentroids: int, return_scores: bool = False, return_centroids: bool = False, niter: int = 20, verbose: bool = False) -> pd.DataFrame \| tuple[pd.DataFrame, np.ndarray]` | `sem_ops/sem_cluster_by.py:58` | `pd.DataFrame` | Clustering bang faiss k-means tren embeddings. Them cot `cluster_id`. |
| `llm_as_judge` | `__call__(self, judge_instruction: str, response_format: BaseModel \| None = None, n_trials: int = 1, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_raw_outputs: bool = False, return_explanations: bool = False, suffix: str = "_judge", examples: pd.DataFrame \| None = None, cot_reasoning: list[str] \| None = None, strategy: ReasoningStrategy \| None = None, extra_cols_to_include: list[str] \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Evaluating", **model_kwargs: Any) -> pd.DataFrame` | `evals/llm_as_judge.py:188` | `pd.DataFrame` | Danh gia moi row bang LLM. Noi bo goi `sem_map` voi n_trials. Disable cache khi chay. |
| `pairwise_judge` | `__call__(self, col1: str, col2: str, judge_instruction: str, response_format: BaseModel \| None = None, n_trials: int = 1, permute_cols: bool = False, system_prompt: str \| None = None, postprocessor: Callable = map_postprocess, return_raw_outputs: bool = False, return_explanations: bool = False, suffix: str = "_judge", examples: pd.DataFrame \| None = None, cot_reasoning: list[str] \| None = None, strategy: ReasoningStrategy \| None = None, safe_mode: bool = False, progress_bar_desc: str = "Evaluating", **model_kwargs: Any) -> pd.DataFrame` | `evals/pairwise_judge.py:70` | `pd.DataFrame` | So sanh cap-doi 2 columns. Noi bo goi `llm_as_judge`. Ho tro permute_cols de giam position bias. |

## Phan loai theo dependency

### Chi can LM (lotus.settings.lm)
- `sem_filter`, `sem_map`, `sem_join`, `sem_agg`, `sem_topk`, `sem_extract`, `llm_as_judge`, `pairwise_judge`

### Chi can RM + VS (lotus.settings.rm, lotus.settings.vs)
- `sem_search`, `sem_sim_join`, `sem_dedup`, `sem_index`, `sem_cluster_by`

### Khong can model nao
- `load_sem_index`, `sem_partition_by`

### Can ca LM va RM/VS (khi dung cascade)
- `sem_filter` voi `cascade_args` va `proxy_model=ProxyModel.EMBEDDING_MODEL` (sem_filter.py:435-441)
- `sem_join` voi `cascade_args` (goi `sem_sim_join` noi bo) (sem_join.py:746-773)
- `sem_topk` voi `method="quick-sem"` (goi `sem_index` + `sem_search`) (sem_topk.py:782-788)

## Shared infrastructure

Tat ca LLM operators chia se:
- `@operator_cache` decorator (cache.py:33)
- `lotus.nl_expression.parse_cols()` (nl_expression.py:4) de extract column names tu `{col}` syntax
- `lotus.nl_expression.nle2str()` (nl_expression.py:17) de format instruction string
- `task_instructions.df2multimodal_info()` (task_instructions.py:364) de convert DataFrame rows thanh multimodal dicts
