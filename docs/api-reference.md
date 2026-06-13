# API Reference

All public symbols are importable directly from `groundedql`:

```python
import groundedql

# or decompose individually:
from groundedql import execute_query_plan, QueryPlanPlanner, QueryAgent
```

---

## Quick Reference

| Symbol | Kind | Summary |
|---|---|---|
| [`groundedql init`](#groundedql-init) | CLI | Introspect Postgres and generate `schema.yaml`. |
| [`groundedql describe`](#groundedql-describe) | CLI | Enrich schema with LLM-generated descriptions. |
| [`execute_query_plan`](#execute_query_plan) | function | Compile + execute a QueryPlan. Primary entrypoint. |
| [`validate_query_plan`](#validate_query_plan) | function | Validate a plan offline. Returns list of error strings. |
| [`validate_query_plan_dict`](#validate_query_plan_dict) | function | Structured validation. Returns typed error objects. |
| [`load_and_validate_schema`](#load_and_validate_schema) | function | Parse and validate `schema.yaml`. |
| [`QueryPlanPlanner`](#queryplanplanner) | class | LLM → QueryPlan orchestration with retry loop (legacy pipeline). |
| [`QueryAgent`](#queryagent) | class | Natural language → executed result in one call (intent pipeline). |
| [`IntentPlanner`](#intentplanner) | class | Two-stage intent extraction → plan builder pipeline. |
| [`IntentMemory`](#intentmemory) | class | ChromaDB-backed few-shot memory for consistency. |
| [`build_value_index`](#build_value_index) | function | Query DB for distinct values of categorical columns. |
| [`normalize_intent`](#normalize_intent) | function | Deterministic intent canonicalization. |
| [`QueryPlan`](#queryplan) | Pydantic model | Type-safe QueryPlan construction. |
| [`queryplan_json_schema`](#queryplan_json_schema) | function | JSON Schema for structured LLM output. |
| [`semantic_lint`](#semantic_lint) | function | Run semantic checks without execution. |
| [`auto_inject_joins`](#auto_inject_joins) | function | Auto-inject join paths into a plan. |
| [`build_spec` / `write_spec`](#build_spec) | functions | Programmatic spec builder. |
| [`get_queryplan_instructions`](#get_queryplan_instructions) | function | Build full LLM system prompt string. |
| [`Compiler`](#compiler-low-level) | class | Low-level: compile a plan to `(sql, params)`. |

---

## CLI Commands

GroundedQL ships with a command-line interface for schema management. After `pip install groundedql`, the `groundedql` command is available.

### `groundedql init`

Introspect a Postgres database and generate `schema.yaml` with full structure.

```bash
groundedql init --db "postgresql://user:pass@host/db" [options]
```

**Options:**

| Flag | Default | Description |
|---|---|---|
| `--db` | required | Database URL (e.g., `postgresql://user:pass@host/db`) |
| `--output`, `-o` | `config/schema.yaml` | Output path for the generated schema |
| `--schema` | `public` | Postgres schema to introspect |
| `--exclude` | `[]` | Table names to skip (space-separated) |

**Auto-detects:**

- Tables, columns, and types (mapped to GroundedQL types)
- Physical names with proper quoting for camelCase identifiers
- `primary_id` from primary key constraints
- `primary_date` via heuristic (looks for `created_at`, `entry_date`, etc.)
- `keyword_search_or` for tables with multiple text columns named like `description`, `name`, etc.
- `links` from foreign key constraints (with `left` join type)

**Example:**

```bash
groundedql init \
    --db "postgresql://user:pass@host/db" \
    --schema public \
    --exclude migrations audit_log \
    -o config/schema.yaml

# Schema written to config/schema.yaml
#   13 tables, 183 columns, 4 links
```

### `groundedql describe`

Enrich an existing `schema.yaml` with LLM-generated table and column descriptions.

```bash
export OPENAI_API_KEY=sk-...
groundedql describe --schema config/schema.yaml [--db URL]
```

**Options:**

| Flag | Default | Description |
|---|---|---|
| `--schema` | `config/schema.yaml` | Path to the schema file to enrich |
| `--db` | `None` | Database URL for sampling values (optional, improves descriptions) |
| `--api-key` | env var | LLM API key (reads `LLM_API_KEY` or `OPENAI_API_KEY` from env if omitted) |
| `--base-url` | `https://api.openai.com/v1` | Any OpenAI-compatible API endpoint |
| `--model` | `gpt-4o-mini` | Model name to use |

**LLM-agnostic** — works with any provider that exposes an OpenAI-compatible `/v1/chat/completions` endpoint. No LLM SDK is required; GroundedQL uses raw HTTP.

When `--db` is provided, the command samples distinct values from each column and includes them in the prompt, producing significantly better descriptions (e.g., the LLM can detect that "values are in UPPER CASE" or "contains free-text descriptions").

**Examples:**

```bash
# OpenAI (default)
export LLM_API_KEY=sk-...
groundedql describe --schema config/schema.yaml --db "postgresql://user:pass@host/db"

# Groq (free)
groundedql describe \
    --api-key gsk_... \
    --base-url https://api.groq.com/openai/v1 \
    --model llama-3.1-70b-versatile \
    --schema config/schema.yaml

# Ollama (local)
groundedql describe \
    --api-key ollama \
    --base-url http://localhost:11434/v1 \
    --model llama3 \
    --schema config/schema.yaml
```

### Programmatic access

Both commands are also available as Python functions:

```python
from groundedql.cli import introspect_database, describe_schema

schema = introspect_database(db_url="postgresql://...", schema_name="public")
describe_schema(schema_path="config/schema.yaml", db_url="postgresql://...")
```

---

## `execute_query_plan`

The primary entrypoint. Compiles and executes a QueryPlan against a live Postgres database.

```python
def execute_query_plan(
    *,
    engine: Engine,
    schema_path: str,
    query_plan: Dict[str, Any],
    raise_on_error: bool = False,
    statement_timeout_ms: int = 30_000,
) -> Dict[str, Any]
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `engine` | `sqlalchemy.Engine` | required | A SQLAlchemy engine connected to your Postgres DB |
| `schema_path` | `str` | required | Path to your `schema.yaml` file |
| `query_plan` | `dict` | required | A QueryPlan dict (see [QueryPlan Reference](query-plan-reference.md)) |
| `raise_on_error` | `bool` | `False` | If `True`, raises typed exceptions instead of returning an error dict |
| `statement_timeout_ms` | `int` | `30_000` | Per-query timeout in milliseconds |

**Returns — success:**

```python
{
    "rows":      [{"col": val, ...}, ...],  # list of row dicts
    "row_count": 42,                        # number of rows returned
    "columns":   ["col1", "col2", ...],     # column names in order
    "sql":       "SELECT ...",              # compiled SQL (for logging/debug)
    "params":    {"p0": "Germany", ...},    # bind parameter values
    "meta":      {...},                     # forwarded from plan["meta"] if present
}
```

**Returns — error** (when `raise_on_error=False`):

```python
{"error": {"message": "..."}}
```

**Execution steps:**

1. Load and validate `schema.yaml`
2. Strip `meta` key from plan (forwarded to output, not compiled)
3. Resolve `$relative_date` sentinels to concrete UTC timestamps
4. `auto_inject_joins` — insert shortest join path for multi-table refs
5. `Compiler(schema).compile(plan)` → `(sql, params)`
6. `Executor(engine, timeout).execute(sql, params)` → result dict

**Example:**

```python
from sqlalchemy import create_engine
import groundedql

engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

result = groundedql.execute_query_plan(
    engine=engine,
    schema_path="config/schema.yaml",
    query_plan={
        "dataset": "orders",
        "filters": [{"field": "ship_country", "op": "=", "value": "Germany"}],
        "metrics": [{"agg": "count", "field": "*", "alias": "n"}],
    },
    raise_on_error=True,
    statement_timeout_ms=10_000,
)
print(result["rows"])
```

---

## `validate_query_plan`

Validates a QueryPlan **offline** — no database connection required.

```python
def validate_query_plan(
    query_plan: Dict[str, Any],
    schema_path: str,
) -> List[str]
```

**Returns:** A list of error strings. Empty list means valid.

```python
errors = groundedql.validate_query_plan(plan, "config/schema.yaml")
if errors:
    for e in errors:
        print(e)
```

Validation runs three checks in order:

1. **Pydantic** — structural schema (required fields, valid operators/aggs, `extra="forbid"`)
2. **Allowlist** — every table and column present in `schema.yaml`
3. **Semantic rules** — rollup `field` references inner aliases, metric aliases are unique

---

## `load_and_validate_schema`

Load `schema.yaml` and run validation.

```python
def load_and_validate_schema(schema_path: str) -> Dict[str, Any]
```

**Returns:** Parsed schema dict.  
**Raises:** `SchemaError` on fatal issues. Prints warnings to stdout for non-fatal issues.

```python
schema = groundedql.load_and_validate_schema("config/schema.yaml")
```

---

## `QueryPlanPlanner`

LLM → QueryPlan orchestration with validation and retry.

```python
class QueryPlanPlanner:
    def __init__(
        self,
        *,
        llm: Any,
        schema_path: str,
        spec_path: str | None = None,
        spec_dict: dict | None = None,
        temperature: float = 0.0,
        model: str | None = None,
    )
```

**Parameters:**

| Parameter | Description |
|---|---|
| `llm` | Any LLM — OpenAI client, LangChain model, callable, or `LLMClient` instance |
| `schema_path` | Path to `schema.yaml` |
| `spec_path` | Path to `queryplan_spec_generated.yaml`. Auto-generates from schema if omitted. |
| `spec_dict` | Pre-parsed spec dict. Takes priority over `spec_path`. |
| `temperature` | LLM temperature. Use `0.0` for maximum determinism. |
| `model` | Model name hint (used only by the OpenAI adapter). |

### `plan(question)` — Single Shot

One LLM call, auto-fixes applied. No validation, no retry. Fastest path.

```python
plan = planner.plan("How many customers are in France?")
```

### `plan_with_retry(question, max_retries=1)` — Full Pipeline

LLM call → auto-fix → validate → semantic lint → retry with structured feedback if needed.

```python
plan = planner.plan_with_retry(
    "Top 5 products by revenue last 30 days",
    max_retries=2,
)
```

The returned plan includes a `meta` dict:

```python
plan["meta"] == {
    "plan_hash":          "a3f1b2c9...",  # SHA-256 fingerprint (stable across calls)
    "retry_count":        0,              # 0 = first attempt succeeded
    "auto_fixes_applied": [],             # list of applied auto-fix descriptions
    "validation_errors":  [],             # empty = valid
    "lint_errors":        [],             # semantic lint; QueryAgent refuses to execute if non-empty (when enforce_semantic_lint=True)
}
```

---

## `QueryAgent`

High-level wrapper: natural language question → executed result in one call. Uses the two-stage intent pipeline by default.

```python
class QueryAgent:
    def __init__(
        self,
        *,
        engine: Engine,
        schema_path: str,
        spec_path: str | None = None,
        llm: Any,
        max_plan_retries: int = 2,
        enforce_semantic_lint: bool = True,
        use_intent_pipeline: bool = True,
    )
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `engine` | `sqlalchemy.Engine` | required | SQLAlchemy engine connected to Postgres |
| `schema_path` | `str` | required | Path to `schema.yaml` |
| `spec_path` | `str \| None` | `None` | Path to spec file. Auto-generates from schema if omitted. |
| `llm` | `Any` | required | Any LLM — OpenAI client, LangChain model, callable |
| `max_plan_retries` | `int` | `2` | Extra LLM attempts for the legacy pipeline fallback |
| `enforce_semantic_lint` | `bool` | `True` | Block execution on lint errors (legacy pipeline only) |
| `use_intent_pipeline` | `bool` | `True` | Use two-stage intent pipeline; set `False` for legacy |

**On initialization**, `QueryAgent` automatically:

1. **Builds a value index** — queries the database for distinct values of categorical columns
2. **Initializes IntentMemory** — connects to ChromaDB at `<schema_dir>/.intent_memory/`
3. **Creates an IntentPlanner** — configured with the value index and memory

### `ask(question)`

```python
result = agent.ask("What are the top 10 countries by number of orders?")
```

**Async frameworks:** the API is synchronous only. From FastAPI / Starlette, call `ask` via `asyncio.to_thread` or `run_in_executor` so the event loop is not blocked. See [Async web apps](getting-started.md#async-apps) in Getting Started. Native `AsyncEngine` / `async def ask` support may be added in a future release.

Returns the same structure as `execute_query_plan` on success. On plan validation failure after all retries:

```python
{
    "error": {
        "message":          "validation failed after retries",
        "validation_errors": [{"path": "$.metrics[0].field", "message": "..."}],
        "plan":             {...},
    }
}
```

When semantic lint still fails (and `enforce_semantic_lint` is True):

```python
{
    "error": {
        "message":     "QueryPlan failed semantic lint after retries — plan does not match the question.",
        "lint_errors": ["Lint: ...", ...],
        "plan":        {...},
    }
}
```

---

## `IntentPlanner`

Two-stage planner: LLM intent extraction → deterministic plan builder. This is the core of GroundedQL's consistency pipeline.

```python
from groundedql.intent_planner import IntentPlanner

class IntentPlanner:
    def __init__(
        self,
        *,
        llm: Any,
        schema_path: str,
        temperature: float = 0.0,
        model: str | None = None,
        value_index: dict | None = None,
        max_intent_retries: int = 2,
        memory: IntentMemory | None = None,
    )
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `llm` | `Any` | required | LLM client or callable |
| `schema_path` | `str` | required | Path to `schema.yaml` |
| `value_index` | `dict` | `None` | Pre-built value index from `build_value_index()` |
| `max_intent_retries` | `int` | `2` | Retries when value validation fails |
| `memory` | `IntentMemory` | `None` | Few-shot memory instance |

### `plan(question)` — Full Pipeline

Runs the complete intent pipeline:

1. Retrieve few-shot examples from memory
2. LLM extracts intent (with value index + examples in prompt)
3. Normalize intent deterministically
4. Validate against value index, retry with feedback if needed
5. Build QueryPlan from validated intent
6. Store successful (question, intent) pair in memory

```python
plan = intent_planner.plan("how many plumbing issues in bechtel last year?")
```

The returned plan includes a `meta` dict:

```python
plan["meta"] == {
    "pipeline":           "intent",
    "intent":             {...},    # the normalized intent
    "retry_count":        0,
    "auto_fixes_applied": [],
    "validation_errors":  [],
    "lint_errors":        [],
}
```

---

## `IntentMemory`

ChromaDB-backed storage of (question, intent) pairs for few-shot prompting.

```python
from groundedql.intent_memory import IntentMemory

class IntentMemory:
    def __init__(
        self,
        persist_directory: str | None = None,
        collection_name: str = "intent_memory",
        max_examples: int = 500,
    )
```

**Parameters:**

| Parameter | Type | Default | Description |
|---|---|---|---|
| `persist_directory` | `str` | `~/.groundedql/intent_memory` | ChromaDB storage directory |
| `collection_name` | `str` | `"intent_memory"` | ChromaDB collection name |
| `max_examples` | `int` | `500` | Maximum stored examples (oldest evicted) |

Falls back gracefully if `chromadb` is not installed — memory is simply disabled.

### `store(question, intent)`

Store a successful (question, intent) pair. Deduplicates by question hash.

```python
memory.store("how many plumbing issues?", {"dataset": "work_orders", ...})
```

### `retrieve(question, top_k=3, min_similarity=0.60)`

Find similar past questions. Returns list of `{"question", "intent", "similarity"}`.

```python
examples = memory.retrieve("how many electrical problems?")
# [{"question": "how many plumbing issues?", "intent": {...}, "similarity": 0.89}]
```

### `format_few_shot_examples(examples)`

Format retrieved examples as a prompt section for the LLM.

```python
prompt_section = memory.format_few_shot_examples(examples)
```

---

## `build_value_index`

Query the database for distinct values of categorical columns.

```python
from groundedql.value_index import build_value_index

def build_value_index(
    engine: Engine,
    schema_path: str,
    max_distinct: int = 500,
) -> Dict[str, Dict[str, List[str]]]
```

**Returns:** Nested dict of `{table: {column: [values]}}`.

```python
index = build_value_index(engine, "config/schema.yaml")
# {"orders": {"ship_country": ["Germany", "France", ...], ...}}
```

Built once at `QueryAgent` startup, shared across all sessions. Related functions:

| Function | Description |
|---|---|
| `format_value_index_for_prompt(index)` | Format as compact string for LLM prompt injection |
| `fuzzy_resolve(value, known_values)` | Find closest match using substring + difflib |
| `resolve_intent_values(intent, index)` | Fuzzy-correct all filter values in an intent |
| `validate_intent_against_index(intent, index)` | Check filter values exist; returns list of issues |

---

## `normalize_intent`

Deterministic canonicalization of LLM-extracted intents.

```python
from groundedql.intent_normalize import normalize_intent

def normalize_intent(
    intent: Dict[str, Any],
    schema: Dict[str, Any],
) -> Dict[str, Any]
```

Applies these rules:

1. **Absorb keyword filters** — if the LLM put the keyword as both `keyword` and a filter on a `keyword_search_or` column, remove the redundant filter
2. **Ensure group_by is a list** — normalize from string/null to list
3. **Auto group_by for multi-value filters** — if a filter has multiple values with count/sum/avg aggregation, add the column to group_by

```python
intent = normalize_intent(raw_intent, schema)
```

---

## `validate_query_plan_dict`

Lower-level validation that returns structured error objects.

```python
def validate_query_plan_dict(
    plan_dict: Dict[str, Any],
    schema_path: str,
) -> Tuple[Optional[QueryPlan], List[ValidationErrorItem]]
```

**Returns:** `(parsed_plan or None, list of ValidationErrorItem)`

```python
from groundedql import validate_query_plan_dict

plan, errors = validate_query_plan_dict(plan_dict, "config/schema.yaml")
for e in errors:
    print(f"{e.path}: {e.message}")
```

`ValidationErrorItem` fields:

| Field | Type | Description |
|---|---|---|
| `path` | `str` | JSONPath-style location, e.g. `"$.filters[0].field"` |
| `message` | `str` | Human-readable error description |

---

## `QueryPlan`

The Pydantic model for type-safe plan construction:

```python
from groundedql import QueryPlan

plan = QueryPlan(
    dataset="orders",
    filters=[{"field": "ship_country", "op": "=", "value": "Germany"}],
    metrics=[{"agg": "count", "field": "*", "alias": "n"}],
)

result = groundedql.execute_query_plan(
    engine=engine,
    schema_path="config/schema.yaml",
    query_plan=plan.model_dump(),
)
```

---

## `queryplan_json_schema`

Returns the JSON Schema for the QueryPlan model. Pass this to any LLM that supports structured output natively:

```python
from groundedql import queryplan_json_schema

schema = queryplan_json_schema()
# {"type": "object", "properties": {...}, ...}
```

---

## `semantic_lint`

Run semantic lint checks against a plan without executing it:

```python
def semantic_lint(
    question: str,
    plan: Dict[str, Any],
    schema: Dict[str, Any] | None = None,
) -> List[str]
```

```python
from groundedql import semantic_lint

errors = semantic_lint(
    question="Top 10 customers by revenue",
    plan=plan_dict,
    schema=schema_dict,  # optional — enables grain checks
)
for e in errors:
    print(e)
```

---

## `auto_inject_joins`

Lower-level join utilities (`auto_inject_joins`, `build_link_graph`, `shortest_join_path`) for direct pipeline control:

```python
from groundedql import auto_inject_joins, build_link_graph, shortest_join_path

# Build the adjacency graph from schema links
graph = build_link_graph(schema)

# Find shortest path between two tables
path = shortest_join_path(graph, start="order_details", end="customers")
# Returns list of link defs in traversal order, or None if no path

# Auto-inject joins into a plan
enriched_plan = auto_inject_joins(plan_dict, schema)
```

---

## `build_spec`

Programmatic spec builder (equivalent to the CLI command):

```python
from groundedql import build_spec, write_spec

spec = build_spec("config/schema.yaml")          # returns dict
write_spec(spec, "config/queryplan_spec_generated.yaml")
```

---

## `get_queryplan_instructions`

Build the full LLM system prompt string from spec + schema:

```python
from groundedql.api.spec_api import get_queryplan_instructions

prompt = get_queryplan_instructions(
    schema_path="config/schema.yaml",
    spec_path="config/queryplan_spec_generated.yaml",
    include_schema_yaml=True,   # append raw schema.yaml — recommended
)
```

Useful when building your own message lists instead of using `QueryPlanPlanner`.

---

## `Compiler` (low-level)

Direct access to the compiler if you manage the pipeline yourself:

```python
from groundedql.compiler import Compiler

schema = groundedql.load_and_validate_schema("config/schema.yaml")
compiler = Compiler(
    schema,
    default_limit=100,
    max_limit=1000,
    max_joins=8,
)

sql, params = compiler.compile(plan_dict)
print(sql)
print(params)
```

`compile()` returns `(sql_string, params_dict)` using named bind parameters compatible with psycopg2 (`%(name)s` style).

---
