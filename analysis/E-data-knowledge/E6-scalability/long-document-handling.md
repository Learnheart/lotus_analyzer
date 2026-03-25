# E6 - Long Document Handling

## Tổng quan

LOTUS xử lý long documents qua hai strategies: TRUNCATE (cắt bớt) và CHUNK (split column lớn nhất). Chủ yếu dùng trong sem_agg, với hierarchical tree handling cho multiple documents.

---

## 1. LongContextStrategy Enum

**Location**: `types.py:134-138`

```python
class LongContextStrategy(Enum):
    TRUNCATE = auto()
    CHUNK = auto()
```

Default trong sem_agg: `LongContextStrategy.CHUNK` (`sem_agg.py:362`)

---

## 2. TRUNCATE Strategy

**Location**: `long_context_strategy.py:85-142`

```python
def _create_truncated_documents(df, cols, model, extra_tokens):
    max_doc_tokens = model.max_ctx_len - model.max_tokens - extra_tokens  # :96
```

### Process
1. Tính token budget: `max_ctx_len - max_tokens - extra_tokens` (`long_context_strategy.py:96`)
2. Cho mỗi document, check nếu vượt budget (`long_context_strategy.py:110`)
3. Nếu vượt:
   - Tính ellipsis tokens (`long_context_strategy.py:116`)
   - `available_tokens = max_doc_tokens - ellipsis_tokens` (`long_context_strategy.py:119`)
   - Encode → truncate tokens → decode (`long_context_strategy.py:126-132`)
   - Append "..." (`long_context_strategy.py:133`)

### Token-accurate truncation
```python
tokens = model.encode_text(doc_str)                    # :126
truncated_tokens = tokens[:available_tokens]           # :129
truncated_text = model.decode_tokens(truncated_tokens) # :132
truncated_docs.append(truncated_text + ellipsis)       # :133
```

Cắt chính xác theo token boundary, không theo character.

---

## 3. CHUNK Strategy

**Location**: `long_context_strategy.py:145-233`

```python
def _create_chunked_documents(df, cols, model, extra_tokens):
```

### Process cho mỗi row

1. Tạo document string (`long_context_strategy.py:164`)
2. Check nếu fits (`long_context_strategy.py:167`)
3. Nếu không fit:
   a. Tìm column có nhiều tokens nhất:
      ```python
      for col in cols:
          col_tokens = model.count_tokens(str(row[col]))   # :181
          if col_tokens > max_tokens_count:
              max_tokens_col = col                          # :185
      ```
   b. Tạo document với column lớn nhất emptied:
      ```python
      row_copy[max_tokens_col] = ""                        # :192
      doc_str_emptied = df2text(pd.DataFrame([row_copy]), cols)[0]  # :193
      ```
   c. Tính available tokens cho column lớn nhất:
      ```python
      available_tokens = max_doc_tokens - doc_str_emptied_tokens  # :197
      ```
   d. Split column content:
      ```python
      chunks = _split_text_by_tokens(max_col_content, available_tokens, model)  # :210
      ```
   e. Tạo chunked docs (duplicate other columns per chunk):
      ```python
      for chunk_idx, chunk in enumerate(chunks):
          row_copy[max_tokens_col] = chunk                  # :215
          chunk_doc_str = df2text(pd.DataFrame([row_copy]), cols)[0]  # :216
      ```

---

## 4. _split_text_by_tokens

**Location**: `long_context_strategy.py:236-261`

```python
def _split_text_by_tokens(text, max_tokens, model):
    tokens = model.encode_text(text)           # :249
    if len(tokens) <= max_tokens:
        return [text]                           # :252
    chunks = []
    for i in range(0, len(tokens), max_tokens):  # :256
        chunk_tokens = tokens[i : i + max_tokens]  # :257
        chunk_text = model.decode_tokens(chunk_tokens)  # :258
        chunks.append(chunk_text)
    return chunks
```

Exact token-based splitting:
- Encode → split at token boundaries → decode
- Không có overlap giữa chunks
- Có thể cắt giữa words/sentences

---

## 5. sem_agg Hierarchical Tree cho Multiple Documents

Khi tổng documents vượt context window, sem_agg tự động tạo tree:

**Location**: `sem_agg.py:164-219`

```python
while len(doc_list) != 1 or summaries == []:
    # ... fit docs into context window ...
    if (new_tokens + context_tokens + template_tokens >
        model.max_ctx_len - model.max_tokens):          # :183
        # Close batch, start new one
    # ...
    lm_output = model(batch, ...)                        # :211
    doc_list = summaries                                 # :217 - summaries become next level input
    tree_level += 1                                      # :219
```

Tree levels:
```
Level 0: documents → summaries (leaf template)
Level 1: summaries → meta-summaries (node template)
Level N: ... → final answer
```

---

## 6. Long Context trong sem_agg Integration

**Location**: `sem_agg.py:411-428`

```python
if long_context_strategy in (LongContextStrategy.TRUNCATE, LongContextStrategy.CHUNK):
    leaf_template = _get_leaf_instruction_template(formatted_usr_instr)    # :415
    template_tokens = lotus.settings.lm.count_tokens(leaf_template)       # :416
    docs_input = create_chunked_documents(
        self._obj, col_li, lotus.settings.lm,
        long_context_strategy, template_tokens                            # :418-419
    )
```

`template_tokens` được trừ khỏi budget → đảm bảo template + document fit trong context.

---

## 7. Kết luận

Long document handling trong LOTUS:
- **TRUNCATE**: Simple, lossy, token-accurate
- **CHUNK**: Intelligent, splits largest column, duplicates others
- **Tree aggregation**: Handles many documents via progressive summarization
- **No overlap**: Chunks không overlap
- **No sentence-aware splitting**: Có thể cắt giữa câu
- **Default CHUNK**: Cho sem_agg
