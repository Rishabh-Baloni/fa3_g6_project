# What llm.py does

`backend/llm.py` is the single gateway for every model call. It turns a role (`translation`, `rag`, `reasoning`) plus a list of messages into text or JSON, and a list of strings into embedding vectors. It also owns the Redis caches and loads the prompt templates.

Main parts:

1. Redis cache helper: `cache_key()` and `Cache`
2. LangChain LLM cache in Redis: `RedisLlmCache`
3. The gateway: `ModelRole`, errors, `ModelStatus`, `LlmGateway`
4. JSON extraction: `parse_json_object()`
5. Prompt and persona loading: `persona()`, `render()`, `messages()`
6. Connectivity self-test: `check_models()` (`python -m llm`)

---

## Overall flow

```
 caller (workflow / validation / rag / query / main)
     │  messages("auditor", "report_summary", **values)      ◄── prompts/personas.yaml, prompts/prompts.yaml
     ▼
 LlmGateway.complete(role, messages)
     │  chat_model(role)  ──►  _build_chat()  ──►  ChatBedrockConverse | ChatGroq   (temperature 0, cache=RedisLlmCache)
     │                                              │
     │                                              ├── cache hit  ──► Redis  llm:{sha256(llm_string + prompt)}
     │                                              └── cache miss ──► provider call ──► store in Redis (TTL)
     ▼
 text  ──►  parse_json_object()  ──►  dict

 LlmGateway.embed(texts) ──► batches of embedding_batch_size ──► BedrockEmbeddings | HuggingFaceEndpointEmbeddings
```

Everything in Redis has an expiry and a durable source elsewhere. A Redis outage makes the system slower, not wrong: read and write errors are logged and treated as cache misses.

---

## 1. `cache_key` and `Cache`

| Name | Input | Output |
|---|---|---|
| `cache_key(namespace, *parts)` | A namespace string and any JSON-serialisable parts | `"{namespace}:{sha256 of the JSON of parts}"`. Parts are serialised with `sort_keys=True`, so the key is stable |
| `Cache(url)` | Redis URL | Client with 3 s connect and read timeouts, `decode_responses=True` |
| `Cache.ping()` | none | Raises if Redis is unreachable (used for startup checks) |
| `Cache.get_json(key)` | Key | Decoded value, or `None` on a miss or a `RedisError` |
| `Cache.set_json(key, value, ttl_seconds)` | Key, value, TTL | Stores JSON with expiry; errors are logged only |
| `Cache.delete_namespaces(namespaces)` | Tuple like `("llm", "rag")` | Number of keys deleted by scanning `"{namespace}:*"`; stops early on a Redis error |
| `Cache.client` | none | The underlying `redis.Redis` |

## 2. `RedisLlmCache`

A LangChain `BaseCache` attached to every chat model the gateway builds.

| Method | Purpose |
|---|---|
| `key(prompt, llm_string)` | `"llm:" + sha256(llm_string + "\n" + prompt)`. `llm_string` encodes model settings, so a different model or temperature never shares an entry |
| `lookup(...)` | Reads Redis and rebuilds generations with `langchain_core.load.load`. An unreadable entry is discarded and treated as a miss. The beta warning is suppressed |
| `update(...)` | Stores `dumpd(generation)` JSON with the TTL (`llm_cache_ttl_seconds` from settings). Errors are logged |
| `clear()` | Deletes every `llm:*` key |

Because the cache is attached to the model itself, identical prompts from the agent and from the ragas evaluator are answered from Redis without calling the provider.

## 3. The gateway

| Item | Purpose |
|---|---|
| `ModelRole` | `StrEnum` with `TRANSLATION`, `RAG`, `REASONING` |
| `ModelNotConfiguredError` | A feature needs a model that is not set up |
| `ModelCallError` | The provider failed or returned something unusable |
| `ModelStatus` | Frozen dataclass of four strings (`translation`, `rag`, `reasoning`, `embedding`) with `as_dict()` |
| `_to_messages(messages)` | Converts `{"role": "system"|"user", "content": ...}` dicts to LangChain messages |

`LlmGateway(settings, llm_cache)` methods:

| Method | Input | Output |
|---|---|---|
| `is_configured(role)` | `ModelRole` | True when `llm_provider` and that role's model ID are both non-empty |
| `embeddings_configured` (property) | none | True when the provider is set and a model or API base is set |
| `status()` | none | `ModelStatus` with labels like `groq:model-id` or `not configured` |
| `chat_model(role, *, shared=True)` | Role | A `BaseChatModel`. Raises `ModelNotConfiguredError` if not configured or if building fails. `shared=True` caches one instance per role; `shared=False` builds a fresh one (ragas changes the temperature of the model it is given) |
| `_build_chat(model_id)` | Model ID | `ChatBedrockConverse` (with read timeout and `retries` of `llm_max_retries + 1` attempts) or `ChatGroq` (requires `llm_api_key`). Temperature is always 0.0. Provider SDKs are imported lazily inside the branch |
| `embeddings()` | none | Cached `BedrockEmbeddings` or `HuggingFaceEndpointEmbeddings`. For Hugging Face, `embedding_api_base` takes precedence over `embedding_model` |
| `complete(role, messages, *, max_tokens=1500)` | Role, message dicts | Response text. Handles list-style content. Raises `ModelCallError` on provider errors or an empty reply |
| `complete_json(role, messages, **kwargs)` | Same | `complete()` followed by `parse_json_object()` |
| `embed(texts)` | List of strings | List of float vectors, embedded in batches of `max(1, embedding_batch_size)`. Raises `ModelCallError` if a call fails or the vector count differs from the input count |

Why lazy imports and a role-based gateway: nothing outside this file names a model or provider, so switching from Groq to Bedrock is a settings change.

## 4. `parse_json_object(text)`

- Input: raw model output. Output: a `dict`.
- It takes the first `{ ... }` block (greedy regex with `DOTALL`), so code fences and surrounding prose are tolerated.
- Raises `ModelCallError` when no block is found, the JSON is invalid, or the value is not an object.

## 5. Prompts and personas

| Function | Input | Output |
|---|---|---|
| `_load(name)` | YAML file name in `prompts_dir` | Cached dict |
| `persona(role)` | Key in `personas.yaml` | Stripped persona text |
| `render(template_name, **values)` | Key in `prompts.yaml` and values | A `string.Template` filled with `substitute`; `None` values become `""`; result is stripped |
| `messages(role, template_name, **values)` | Persona key, template key, values | `[{"role": "system", ...}, {"role": "user", ...}]` |

`Template.substitute` raises `KeyError` if a template placeholder has no value, so a missing value surfaces during development instead of silently producing a broken prompt.

## 6. `check_models()` and `python -m llm`

- Sends the prompt "Reply with the single word: ready" (max 10 tokens) per configured chat role, then one embedding of `"invoice audit"`, and prints success or failure with timing.
- Uses `LlmGateway(get_settings(), llm_cache=None)` so it always reaches the provider.
- Returns 1 if any call failed, else 0. The `__main__` block ends with `os._exit(code)` to skip interpreter teardown, because some provider SDKs print harmless shutdown tracebacks.

---

## End-to-end example

The translator in `workflow.py` translates a German invoice chunk:

1. `Translator` calls `gateway.complete(ModelRole.TRANSLATION, messages("translator", "translate", source_language="German", text=chunk), max_tokens=4000)`.
2. `messages()` loads the `translator` persona and renders the `translate` template into a system and a user message.
3. `chat_model(TRANSLATION)` builds a `ChatGroq` (or Bedrock) model once, with temperature 0 and the Redis LLM cache attached.
4. The model computes `llm:{sha256(...)}`. First time: a cache miss, so the provider is called and the answer is stored for `llm_cache_ttl_seconds`.
5. The text reply goes back to `Translator`, which calls `parse_json_object()` to read `{"translation": ..., "confidence": ...}`.
6. The same chunk later gets the cached answer without a provider call. If Redis is down, the lookup logs a warning and the provider is called as normal.
