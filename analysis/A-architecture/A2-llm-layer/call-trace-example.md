# A2 - Call Trace Example: sem_filter

## Trace: `df.sem_filter("the {title} is about AI")`

### Gia dinh:
```python
import lotus
from lotus.models import LM
lotus.settings.configure(lm=LM(model="gpt-4o-mini"))
df = pd.DataFrame({"title": ["Machine Learning 101", "Cooking for Beginners"]})
df.sem_filter("the {title} is about AI")
```

---

### Step 1: SemFilterDataframe.__call__ (sem_filter.py:334)

```
df.sem_filter("the {title} is about AI")
  |
  |-- pandas tao SemFilterDataframe(df) -> self._obj = df
  |-- Goi __call__("the {title} is about AI")
  |-- @operator_cache kiem tra cache truoc (cache.py:33)
  |     |-- hash(self._obj + args + kwargs) -> cache_key
  |     |-- Neu cache hit -> return cached result
  |     |-- Neu cache miss -> tiep tuc
  |
  |-- Kiem tra lotus.settings.lm is not None (sem_filter.py:351)
  |
  |-- parse_cols("the {title} is about AI") -> ["title"]
  |     (nl_expression.py:4, regex: r"(?<!\{)\{(?!\{)(.*?)(?<!\})\}(?!\})")
  |
  |-- Kiem tra "title" in df.columns (sem_filter.py:363-365)
  |
  |-- df2multimodal_info(df, ["title"]) -> multimodal_data
  |     (task_instructions.py:364)
  |     Ket qua:
  |     [
  |       {"text": "[Title]: <<Machine Learning 101>>\n", "image": {}},
  |       {"text": "[Title]: <<Cooking for Beginners>>\n", "image": {}}
  |     ]
  |
  |-- nle2str("the {title} is about AI", ["title"])
  |     -> "the Title is about AI"
  |     (nl_expression.py:17)
```

### Step 2: sem_filter() function (sem_filter.py:24)

```
sem_filter(multimodal_data, lm, "the Title is about AI", default=True)
  |
  |-- Voi moi doc trong multimodal_data:
  |     |-- filter_formatter(model, doc, "the Title is about AI")
  |           (task_instructions.py:87)
  |
  |     Tao messages:
  |     [
  |       {"role": "system", "content":
  |         "The user will provide a claim and some relevant context.\n
  |          Your job is to determine whether the claim is true for the given context.\n
  |          Use the following format to provide your answer:\n
  |          Answer: <Your answer here. The answer should be either True or False>"
  |       },
  |       {"role": "user", "content":
  |         "Context:\n[Title]: <<Machine Learning 101>>\n\n\nClaim: the Title is about AI"
  |       }
  |     ]
  |
  |-- inputs = [messages_for_doc1, messages_for_doc2]
  |
  |-- model(inputs) -> LMOutput
```

### Step 3: LM.__call__ (lm.py:123)

```
LM.__call__(messages=[msg1, msg2])
  |
  |-- all_kwargs = {temperature: 0.0, max_completion_tokens: 512}
  |
  |-- Neu enable_cache:
  |     |-- hash moi message:
  |     |     _hash_messages(msg, kwargs) (lm.py:407)
  |     |     -> sha256(str(model) + str(messages) + str(kwargs))
  |     |-- Kiem tra cache: cache.get(hash)
  |     |-- Tach cached vs uncached
  |
  |-- _process_uncached_messages(uncached_data, kwargs)
  |     (lm.py:215)
  |     |
  |     |-- Khong co rate_limit hay tpm_limit:
  |     |     batch_completion(
  |     |       "gpt-4o-mini",
  |     |       [msg1, msg2],
  |     |       drop_params=True,
  |     |       max_workers=64,
  |     |       temperature=0.0,
  |     |       max_completion_tokens=512
  |     |     )
  |     |     (lm.py:250-251)
  |     |
  |     |-- litellm gui 2 requests song song den OpenAI API
  |     |-- Nhan 2 ModelResponse objects
  |
  |-- _update_stats(response, is_cached=False) cho moi uncached response
  |     (lm.py:451)
  |     |-- calculate_cost_from_response(response) (pricing.py:10)
  |     |     -> litellm.completion_cost(completion_response=response)
  |     |-- Cap nhat physical_usage va virtual_usage
  |     |-- Kiem tra usage limits
  |
  |-- _get_top_choice(response) cho moi response
  |     (lm.py:485)
  |     |-- Kiem tra isinstance(response, (AuthenticationError, OpenAIError))
  |     |-- Return response.choices[0].message.content
  |     |-- Vi du: "Answer: True", "Answer: False"
  |
  |-- Return LMOutput(outputs=["Answer: True", "Answer: False"], logprobs=None)
```

### Step 4: filter_postprocess (postprocessors.py:182)

```
filter_postprocess(["Answer: True", "Answer: False"], model, default=True)
  |
  |-- get_cot_postprocessor(model) (postprocessors.py:102)
  |     |-- model.get_model_name() -> "gpt-4o-mini"
  |     |-- Khong match "deepseek-r1" -> dung cot_postprocessor mac dinh
  |
  |-- cot_postprocessor(llm_answers) (postprocessors.py:12)
  |     |-- Voi "Answer: True":
  |     |     answer_idx = find("Answer:") = 0
  |     |     reasoning = "" (khong co reasoning truoc "Answer:")
  |     |     answer = "True"
  |     |-- Voi "Answer: False":
  |     |     answer = "False"
  |     |-- outputs = ["True", "False"]
  |     |-- explanations = ["", ""]
  |
  |-- process_outputs(answer) cho moi answer:
  |     (postprocessors.py:200)
  |     |-- "True" in "True" -> return True
  |     |-- "False" in "False" -> return False
  |     |-- Neu khong parse duoc -> return default (True)
  |
  |-- Return SemanticFilterPostprocessOutput(
  |     raw_outputs=["Answer: True", "Answer: False"],
  |     outputs=[True, False],
  |     explanations=["", ""]
  |   )
```

### Step 5: Return filtered DataFrame (sem_filter.py:551-593)

```
return_all=False (default):
  |
  |-- ids = [0]  (chi index 0 co output=True)
  |-- new_df = df.iloc[[0]]
  |     -> DataFrame({"title": ["Machine Learning 101"]})
  |
  |-- Preserve index_dirs: new_df.attrs["index_dirs"] = df.attrs.get("index_dirs", None)
  |
  |-- Return new_df

Ket qua cuoi cung:
         title
0  Machine Learning 101
```

## Tong ket call stack

```
df.sem_filter("the {title} is about AI")
  -> SemFilterDataframe.__call__()           sem_filter.py:334
    -> @operator_cache                       cache.py:33
    -> parse_cols()                          nl_expression.py:4
    -> df2multimodal_info()                  task_instructions.py:364
    -> nle2str()                             nl_expression.py:17
    -> sem_filter()                          sem_filter.py:24
      -> filter_formatter()                  task_instructions.py:87
      -> LM.__call__()                       lm.py:123
        -> _hash_messages()                  lm.py:407
        -> _process_uncached_messages()      lm.py:215
          -> batch_completion()              litellm (external)
        -> _update_stats()                   lm.py:451
          -> calculate_cost_from_response()  pricing.py:10
        -> _get_top_choice()                 lm.py:485
      -> filter_postprocess()                postprocessors.py:182
        -> cot_postprocessor()               postprocessors.py:12
    -> DataFrame slicing                     sem_filter.py:551-563
```
