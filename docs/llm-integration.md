# LLM Integration

IntentQL is **LLM-agnostic**. The `make_llm_client()` factory supports built-in provider
strings, compatible OpenAI and LangChain clients, custom callables, and objects implementing
IntentQL's small `generate_json` interface.

<div class="qce-path-grid" markdown>
<a class="qce-path-card" href="getting-started/">
  <span class="qce-path-card__kicker">Step 1</span>
  <span class="qce-path-card__title">Set Up IntentQL</span>
  <span class="qce-path-card__desc">Install package, define `schema.yaml`, and generate `queryplan_spec_generated.yaml`.</span>
  <span class="qce-path-card__cta">Go to getting started -></span>
</a>
<a class="qce-path-card" href="query-plan-reference/">
  <span class="qce-path-card__kicker">Step 2</span>
  <span class="qce-path-card__title">Plan Grammar</span>
  <span class="qce-path-card__desc">Understand the QueryPlan shape your model must produce.</span>
  <span class="qce-path-card__cta">Open QueryPlan reference -></span>
</a>
<a class="qce-path-card" href="exceptions/">
  <span class="qce-path-card__kicker">Step 4</span>
  <span class="qce-path-card__title">Production Errors</span>
  <span class="qce-path-card__desc">Use structured `QueryPlanError` and `DatabaseExecutionError` handling in runtime services.</span>
  <span class="qce-path-card__cta">Open exception reference -></span>
</a>
</div>

---

## How Adapter Detection Works

When you pass an LLM object to `QueryPlanPlanner` or `QueryAgent`, IntentQL calls `make_llm_client(obj)` internally and selects the right adapter:

```
make_llm_client(obj)
      │
      ├─ has generate_json()  →  use as-is      (already an LLMClient)
      │
      ├─ has .responses.create  →  OpenAIResponsesJSONAdapter
      │                             (OpenAI Responses API, structured output)
      │
      ├─ has .invoke()  →  LangChainJSONAdapter
      │                     (any LangChain chat model)
      │
      └─ is callable  →  CallableLLMClient
                          (your own function)
```

You can also import and use adapters directly for fine-grained control.

---

## Supported Providers

=== "Mistral"

    IntentQL includes a dependency-free Mistral adapter that calls the Mistral chat
    completions API directly.

    ```bash
    export MISTRAL_AI=...
    # optional
    export MISTRAL_MODEL=mistral-small-latest
    ```

    ```python
    import intentql

    agent = intentql.QueryAgent(
        engine=engine,
        schema_path="config/schema.yaml",
        llm="mistral",
    )
    ```

    Pin a model inline with `llm="mistral:mistral-small-latest"`. The adapter also reads
    `MISTRAL_BASE_URL`, `MISTRAL_MAX_RETRIES`, and `MISTRAL_RETRY_BASE_SECONDS`.

=== "Ollama (built in)"

    IntentQL includes a direct Ollama adapter for local models. No LangChain dependency is
    required.

    ```bash
    export OLLAMA_MODEL=intentql-gemma4
    # optional
    export OLLAMA_BASE_URL=http://127.0.0.1:11434
    export OLLAMA_NUM_CTX=8192
    ```

    ```python
    import intentql

    agent = intentql.QueryAgent(
        engine=engine,
        schema_path="config/schema.yaml",
        llm="ollama",
    )
    ```

    Pin another local model with `llm="ollama:qwen2.5:7b"`.

=== "OpenAI"

    Pass an OpenAI client when you want to use the Responses API adapter.

    ```bash
    pip install intentql openai
    # Optional: pip install "intentql[memory]"  # few-shot memory (ChromaDB)
    ```

    ```python
    from openai import OpenAI
    import intentql

    planner = intentql.QueryPlanPlanner(
        llm=OpenAI(api_key="sk-..."),
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

    The OpenAI adapter requests structured JSON output and still validates the returned
    object through IntentQL's normal plan-validation path.

    **Specify a different model:**

    ```python
    from intentql.llm_adapters import OpenAIResponsesJSONAdapter

    adapter = OpenAIResponsesJSONAdapter(OpenAI(api_key="sk-..."), model="gpt-4o")
    planner = intentql.QueryPlanPlanner(llm=adapter, ...)
    ```

=== "Google Gemini"

    Connect Gemini through a compatible LangChain model.

    ```bash
    pip install langchain-google-genai
    ```

    ```python
    from langchain_google_genai import ChatGoogleGenerativeAI
    import intentql

    planner = intentql.QueryPlanPlanner(
        llm=ChatGoogleGenerativeAI(
            model="gemini-2.0-flash",
            google_api_key="AIza...",
            temperature=0,
        ),
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

=== "Groq"

    Connect Groq through a compatible LangChain model.

    ```bash
    pip install langchain-groq
    ```

    ```python
    from langchain_groq import ChatGroq
    import intentql

    planner = intentql.QueryPlanPlanner(
        llm=ChatGroq(
            model="llama-3.3-70b-versatile",
            api_key="gsk_...",
            temperature=0,
        ),
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

=== "LangChain (any model)"

    Any `langchain_core` chat model with `.invoke()` works:

    ```bash
    pip install langchain-openai   # or langchain-anthropic, etc.
    ```

    ```python
    from langchain_openai import ChatOpenAI
    import intentql

    planner = intentql.QueryPlanPlanner(
        llm=ChatOpenAI(model="gpt-4o-mini", temperature=0, api_key="sk-..."),
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

    The LangChain adapter parses JSON from the model's text output directly — no tool calling or structured output required — so it works with any model, including open-source ones.

=== "Ollama through LangChain"

    Use this option when your application already standardizes on LangChain. For a smaller
    dependency surface, use the built-in Ollama adapter shown above.

    ```bash
    # Install Ollama from https://ollama.ai, then pull a model
    ollama pull qwen2.5:7b
    pip install langchain-ollama
    ```

    ```python
    from langchain_ollama import ChatOllama
    import intentql

    planner = intentql.QueryPlanPlanner(
        llm=ChatOllama(model="qwen2.5:7b"),
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

    !!! tip "Model recommendations for local use"
        `qwen2.5:7b` and `mistral:7b` follow JSON instructions reliably.
        `llama3.2:3b` is faster but may need `max_retries=2`.

=== "Custom Callable"

    Pass any function that matches this signature:

    ```python
    def my_llm(
        json_schema: dict,
        messages: list[dict],
        temperature: float,
    ) -> dict:
        ...
    ```

    ```python
    import anthropic
    import json
    import intentql

    client = anthropic.Anthropic(api_key="sk-ant-...")

    def claude_generate(json_schema, messages, temperature):
        system = next((m["content"] for m in messages if m["role"] == "system"), "")
        user   = next((m["content"] for m in messages if m["role"] == "user"),   "")
        resp = client.messages.create(
            model="claude-3-5-haiku-latest",
            max_tokens=2048,
            system=system + f"\n\nReturn ONLY JSON matching this schema:\n{json.dumps(json_schema)}",
            messages=[{"role": "user", "content": user}],
        )
        return json.loads(resp.content[0].text)

    planner = intentql.QueryPlanPlanner(
        llm=claude_generate,
        schema_path="config/schema.yaml",
        spec_path="config/queryplan_spec_generated.yaml",
    )
    ```

---

## QueryPlanPlanner

`QueryPlanPlanner` handles the full LLM → QueryPlan cycle:

```python
planner = intentql.QueryPlanPlanner(
    llm=your_llm,
    schema_path="config/schema.yaml",
    spec_path="config/queryplan_spec_generated.yaml",
    temperature=0.0,         # use 0 for determinism
)
```

### `plan()` — Single Shot

One LLM call, auto-fixes applied, no validation, no retry. Fastest path.

```python
plan = planner.plan("How many orders shipped to Germany last week?")
```

Use this when you handle validation yourself, or in a streaming context.

### `plan_with_retry()` — Full Pipeline

The production path. Validates the plan and feeds structured error feedback back to the LLM if needed.

```python
plan = planner.plan_with_retry(
    "Average freight cost per destination country, top 10",
    max_retries=2,  # default: 1
)
```

The plan includes a `meta` dict with full observability:

```python
print(plan["meta"])
# {
#   "plan_hash":            "a3f1b2c9d4e5",  # deterministic SHA-256 fingerprint
#   "retry_count":          0,               # 0 = succeeded on first attempt
#   "auto_fixes_applied":   [],              # e.g. ["forced limit=1 for scalar aggregate"]
#   "validation_errors":    [],              # empty = valid
#   "lint_errors":          [],              # semantic lint warnings (non-fatal)
# }
```

### Auto-Fixes Applied Before Validation

| Condition | Fix applied |
|---|---|
| Filter value is a `$relative_date` sentinel | Resolved to concrete UTC timestamp |
| No dimensions + only scalar aggregations | `limit=1, offset=0` forced |
| Grouped plan with `rollup`, no top-N intent | `limit` removed from inner plan |
| Columns from multiple tables, no joins declared | Shortest BFS join path injected |

---

## QueryAgent

The highest-level API — combines the intent pipeline + compiler + executor in one call. On initialization, it builds a value index from your database and connects to ChromaDB for few-shot memory.

```python
agent = intentql.QueryAgent(
    engine=engine,
    schema_path="config/schema.yaml",
    spec_path="config/queryplan_spec_generated.yaml",
    llm=your_llm,
)

result = agent.ask("What are the top 5 products by revenue this quarter?")
```

**Under the hood**, `ask()` runs the full intent pipeline: memory lookup → LLM intent extraction → normalization → value validation → deterministic plan build → compile → execute → memory store.

If the intent pipeline fails for any reason, it automatically falls back to the legacy QueryPlan pipeline with retries.

**Success response:**

```python
{
    "rows":      [{"product_name": "...", "revenue": 12345.67}, ...],
    "row_count": 5,
    "columns":   ["product_name", "revenue"],
    "sql":       "SELECT product_name, sum(...) ...",
    "params":    {"p0": "2026-01-01", ...},
    "meta":      {"pipeline": "intent", "intent": {...}, "retry_count": 0, ...},
}
```

**Error response** (plan failed validation after all retries):

```python
{
    "error": {
        "message": "validation failed after retries",
        "validation_errors": [
            {"path": "$.metrics[0].field", "message": "unknown column 'revenue' on 'orders'"}
        ],
        "plan": {...},
    }
}
```

---

## What the LLM Receives

### Intent Pipeline (Default)

In the two-stage intent pipeline, the LLM receives a richer prompt with up to 5 messages:

```
[system]  Intent extraction instructions
          "You are an intent extractor. Given a database schema and a user
           question, output a JSON object capturing WHAT the user wants..."

[system]  Database schema summary
          (tables, columns, types, descriptions, metadata)

[system]  Known database values (from value index)
          "KNOWN DATABASE VALUES (use these exact values in filters):
             Table: orders
               ship_country: [Germany, France, ...]"

[system]  Few-shot examples (from memory, if similar questions exist)
          "EXAMPLES — For similar questions, here are the intents that were
           previously extracted and verified correct..."

[user]    The natural language question
```

The value index and few-shot examples are injected dynamically — they're not part of the spec file.

### Legacy Pipeline

In the legacy pipeline, the LLM receives three messages:

```
[system]  IntentQL system instructions
          "Produce ONLY valid JSON. Never write SQL. Use logical names only..."

[system]  Your spec YAML + schema YAML
          (tables, columns, operators, QueryPlan shape, examples)

[user]    The natural language question
```

To inspect the exact legacy prompt:

```python
from intentql.api.spec_api import get_queryplan_instructions

prompt = get_queryplan_instructions(
    schema_path="config/schema.yaml",
    spec_path="config/queryplan_spec_generated.yaml",
)
print(prompt)
```

!!! tip "Keep the spec up to date"
    Regenerate `queryplan_spec_generated.yaml` whenever you add tables or columns to `schema.yaml`. The spec is used by the legacy pipeline and as a fallback.

---

## Retry Feedback Format

When `plan_with_retry` detects errors and has retries remaining, the retry message sent to the LLM looks like this:

```
Your previous QueryPlan had the following issues. Please produce a corrected version.

STRUCTURAL ERRORS (hard failures — must be fixed):
- $.filters[0].field: unknown column 'revenue' on table 'orders'
  Available columns: order_id, customer_id, order_date, freight, ship_country

SEMANTIC WARNINGS (strong suggestions):
- Question asks for "top 10" but order_by is empty and limit is not set
```

Retry behavior depends on the selected model and schema. Measure correction rate on your own
regression set before choosing retry limits for production.
