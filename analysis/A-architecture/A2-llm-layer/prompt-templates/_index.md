# Prompt Templates - Master Index

## Tổng quan

LOTUS sử dụng prompt templates được định nghĩa trong `templates/task_instructions.py` và trong từng operator file. Mỗi operator có system prompt và user prompt riêng, được format bởi các formatter functions.

## Master Table

| Operator | Formatter Function | File:Line | Prompt Type | Few-shot? | CoT? | Output Format |
|---|---|---|---|---|---|---|
| `sem_filter` | `filter_formatter()` | `task_instructions.py:87` | System + User (Claim) | Yes (examples_multimodal_data + examples_answer) | Yes (COT, ZS_COT) | `Answer: True/False` |
| `sem_map` | `map_formatter()` | `task_instructions.py:213` | System + User (Instruction) | Yes (examples + answers) | Yes (COT via `map_formatter_cot`, ZS_COT via `map_formatter_zs_cot`) | Free-form text |
| `sem_join` | `filter_formatter()` (reuses sem_filter) | `task_instructions.py:87` | System + User (Claim) | Yes | Yes (COT, ZS_COT) | `Answer: True/False` |
| `sem_agg` | `_get_leaf_instruction_template()` / `_get_node_instruction_template()` | `sem_agg.py:12` / `sem_agg.py:34` | User only (template-based) | No | No | Free-form text |
| `sem_topk` | `get_match_prompt_binary()` | `sem_topk.py:16` | System + User (Question + Documents) | No | Yes (ZS_COT) | `Document 1` or `Document 2` |
| `sem_extract` | `extract_formatter()` | `task_instructions.py:257` | System + User (Context) | No | Yes (COT, ZS_COT) | JSON with specified fields |
| `sem_search` | N/A | N/A | N/A (vector-based) | N/A | N/A | N/A |
| `sem_sim_join` | N/A | N/A | N/A (embedding-based) | N/A | N/A | N/A |
| `llm_as_judge` | `map_formatter()` (reuses sem_map) | `task_instructions.py:213` | System (judge) + User | Yes | Yes | Free-form text |
| `pairwise_judge` | `map_formatter()` (via llm_as_judge) | `task_instructions.py:213` | System (judge) + User | Yes | Yes | Free-form text |

## Reasoning Strategies

Định nghĩa tại `types.py:241-245`:
```python
class ReasoningStrategy(Enum):
    DEFAULT = auto()
    COT = auto()      # Chain-of-Thought with examples
    ZS_COT = auto()   # Zero-Shot Chain-of-Thought
    FEW_SHOT = auto()
```

### CoT Formatting Helpers (task_instructions.py:11-37)

```python
def cot_formatter(reasoning, answer):
    return f"""Reasoning:\n{reasoning}\n\nAnswer: {answer}"""

def answer_only_formatter(answer):
    return f"""Answer: {answer}"""

def cot_prompt_formatter(reasoning_instructions="", answer_instructions=""):
    # Adds "Let's think step by step..." prefix

def non_cot_prompt_formatter(answer_instructions=""):
    # Adds "Use the following format..." prefix
```

### DeepSeek-specific CoT (task_instructions.py:19-22):
```python
def deepseek_cot_formatter():
    return """Please think through your reasoning step by step, then provide your final answer.
    You must put your reasoning inside the <think></think> tags, then provide your
    final answer after the </think> tag with the format: Answer: your answer."""
```

## Nội dung chi tiết

Xem các file riêng trong thư mục này:
- `sem_filter.md` - Filter prompt template
- `sem_map.md` - Map prompt template
- `sem_join.md` - Join prompt template (reuses filter)
- `sem_agg.md` - Aggregation prompt templates (leaf + node)
- `sem_topk.md` - TopK binary comparison prompt
- `sem_search.md` - Vector search (no LLM prompt)
- `sem_extract.md` - Extraction prompt template
- `sem_sim_join.md` - Similarity join (no LLM prompt)
