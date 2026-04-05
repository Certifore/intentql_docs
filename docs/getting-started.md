# Getting Started

Get from zero to a running IntentQL query in under five minutes.

---

## Prerequisites

- Python ≥ 3.10
- A running Postgres database (local or remote)
- Optional: an LLM API key (OpenAI, Gemini, Groq, etc.) — only needed if you want natural language input
- Optional: `pip install "intentql[memory]"` — persistent few-shot memory (recommended for production)

---

## 1. Install

```bash
pip install intentql
```

For natural-language queries with **persistent few-shot memory** (recommended in production), install the memory extra (includes ChromaDB):

```bash
pip install "intentql[memory]"
```

Optional extras for benchmarks or LangChain-based demos:

```bash
pip install "intentql[benchmark]"   # LangChain + OpenAI / Google — see benchmark/
```

??? note "Install from source"
    ```bash
    git clone https://github.com/Certifore/intentql
    cd intentql
    pip install -e ".[dev]"
    ```

---

## 2. Generate Your Schema

`schema.yaml` is the **allowlist** that tells IntentQL which tables and columns exist. The LLM only sees logical names; the compiler maps them to physical Postgres identifiers.

IntentQL can generate this file automatically by introspecting your database:

```bash
intentql init --db "postgresql://user:pass@host/db"
```

This connects to your Postgres database and generates `config/schema.yaml` with:

- All tables and columns with proper type mappings
- Physical `db_table` / `db_column` names (with quoting for camelCase)
- `primary_id` from primary key constraints
- `primary_date` via heuristic (finds `created_at`, `entry_date`, etc.)
- `keyword_search_or` for tables with multiple text columns
- `links` from foreign key constraints

You can exclude tables and target a specific schema:

```bash
intentql init \
    --db "postgresql://user:pass@host/db" \
    --schema public \
    --exclude migrations audit_log \
    -o config/schema.yaml
```

### Enrich with LLM-generated descriptions (recommended)

The bare schema from `init` has structure but no descriptions. Descriptions dramatically improve LLM accuracy because they tell the model what each column means and how to use it.

```bash
export LLM_API_KEY=sk-...   # or OPENAI_API_KEY
intentql describe --schema config/schema.yaml --db "postgresql://user:pass@host/db"
```

This sends each table's structure and sample values to the LLM, which generates concise descriptions for every table and column. The `--db` flag is optional but recommended — sample values give the LLM much better context (e.g., it can detect "values are in UPPER CASE").

!!! info "`describe` is a strong starting point, not the whole story"
    Auto-generated descriptions are comparable to bootstrapping prompts in other NL→SQL tools: they get you productive quickly, but they **cannot** know your business rules (e.g. “this column is a workflow code, not a trade name,” or “never join these two tables for customer-facing answers”).  
    **Plan for one pass of domain review** on columns that are ambiguous, heavily overloaded, or safety-critical. That is the same kind of tuning you would do with hand-written prompts elsewhere — IntentQL just makes it structured and versionable in `schema.yaml`.

Works with **any OpenAI-compatible provider** — no LLM SDK required:

```bash
# Groq (free tier)
intentql describe --api-key gsk_... --base-url https://api.groq.com/openai/v1 --model llama-3.1-70b-versatile

# Ollama (local)
intentql describe --api-key ollama --base-url http://localhost:11434/v1 --model llama3
```

!!! tip "What to review first"
    After `describe`, prioritize columns where mistakes are costly or common: IDs vs. names, status/category codes vs. free text, dates that mean “entered” vs. “completed,” and any column users ask about using everyday words that don’t match DB values. Add one or two sentences of explicit guidance there — that single pass often matters more than polishing every column.

??? example "Example: generated vs. hand-tuned description"
    **Generated:**
    ```yaml
    - name: shop
      type: bigint
      description: "Unique identifier for each shop, use for shop-specific queries."
    ```
    **Hand-tuned:**
    ```yaml
    - name: shop
      type: bigint
      description: >
        Unique identifier for each shop. For questions about trade type
        (plumbing, electrical), prefer filtering description or work_order_description
        with keyword search rather than filtering on shop ID.
    ```

??? note "Writing schema.yaml by hand"
    You can also create `schema.yaml` manually. Here's the minimal structure:

    ```yaml
    version: 1
    dialect: postgres

    tables:
      - name: orders           # (1)
        db_table: orders       # (2)
        description: "Customer orders"
        primary_id: order_id   # (3)
        primary_date: order_date  # (4)
        columns:
          - { name: order_id,    db_column: order_id,    type: integer }
          - { name: customer_id, db_column: customer_id, type: varchar }
          - { name: order_date,  db_column: order_date,  type: date    }
          - { name: freight,     db_column: freight,     type: numeric }
          - { name: ship_country,db_column: ship_country,type: varchar }

    links:
      - name: orders_to_customers
        from_table: orders
        to_table: customers
        join_type: left
        "on":
          - { left: orders.customer_id, op: "=", right: customers.customer_id }
    ```

    1. **Logical name** — what appears in QueryPlan `dataset` and field refs
    2. **Physical name** — the actual Postgres table name
    3. **Primary key** — used for `count_distinct` and semantic lint
    4. **Primary date** — used for automatic time range filter resolution

    !!! warning "Always quote `\"on\":`"
        `on` is a YAML reserved word; bare `on:` is parsed as `true:`.

See the full [Schema Reference](schema-reference.md) for all options.

---

## 3. Run Your First Query

!!! info "Spec file auto-generated"
    `QueryAgent` automatically generates and manages the LLM spec file (`queryplan_spec_generated.yaml`) from your schema. You don't need to run a separate command — it regenerates whenever `schema.yaml` changes.

### Option A — Hand-written plan (no LLM needed, no schema descriptions needed)

```python title="quickstart.py"
from sqlalchemy import create_engine
import intentql

engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

result = intentql.execute_query_plan(
    engine=engine,
    schema_path="config/schema.yaml",
    query_plan={
        "dataset": "orders",
        "filters": [
            {"field": "ship_country", "op": "=", "value": "Germany"}
        ],
        "metrics": [
            {"agg": "count", "field": "*", "alias": "total_orders"},
            {"agg": "sum",   "field": "freight", "alias": "total_freight"},
        ],
        "limit": 1,
    },
)

print(result["rows"])
# [{"total_orders": 122, "total_freight": 4891.5}]

print(result["sql"])
# SELECT count(*) AS total_orders, sum(freight) AS total_freight
# FROM orders WHERE ship_country = %(p0)s
```

### Option B — Natural language via `QueryAgent`

`QueryAgent` uses the two-stage intent pipeline by default: the LLM extracts a lightweight intent, then deterministic code builds the plan. At startup it also builds a value index from your database and initializes ChromaDB-backed few-shot memory.

```python title="agent_quickstart.py"
from sqlalchemy import create_engine
from openai import OpenAI
import intentql

engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

agent = intentql.QueryAgent(
    engine=engine,
    schema_path="config/schema.yaml",
    llm=OpenAI(api_key="sk-..."),
)

result = agent.ask("Which country had the most orders last month?")
print(result["rows"])    # [{"ship_country": "Germany", "n": 34}]
print(result["sql"])
```

On the first query, the agent logs the value index build and memory initialization:

```
[ValueIndex] orders.ship_country: 21 values
[IntentMemory] Loaded 0 examples from ChromaDB
[IntentMemory] Found 0 similar examples
[DSL] Intent: {"dataset": "orders", "aggregation": "count", ...}
[IntentMemory] Stored example (1 total)
```

As you ask more questions, the memory accumulates verified intents and guides future extractions for consistency.

### Option C — LLM planner, execute yourself

```python title="planner_quickstart.py"
from sqlalchemy import create_engine
from openai import OpenAI
import intentql

engine = create_engine("postgresql+psycopg2://user:pass@localhost/mydb")

planner = intentql.QueryPlanPlanner(
    llm=OpenAI(api_key="sk-..."),
    schema_path="config/schema.yaml",
)

# Step 1: generate a QueryPlan from natural language
plan = planner.plan_with_retry("Top 5 customers by freight spend", max_retries=2)
print(plan["meta"]["retry_count"])       # 0 — succeeded on first attempt
print(plan["meta"]["auto_fixes_applied"])

# Step 2: validate independently (optional — execute_query_plan does this too)
errors = intentql.validate_query_plan(plan, "config/schema.yaml")
assert not errors

# Step 3: execute
result = intentql.execute_query_plan(engine=engine, schema_path="config/schema.yaml", query_plan=plan)
print(result["rows"])
```

---

## 4. Async web apps { #async-apps }

For **FastAPI**, **Starlette**, or any async web stack, keep in mind: IntentQL’s public API is **synchronous**. `QueryAgent.ask`, `execute_query_plan`, and `QueryPlanPlanner` use a standard SQLAlchemy **sync** `Engine` and block for the duration of each call (LLM + database work).

There is **no** `async def ask()` or `AsyncEngine` support in the library yet. That may be added in a future release.

**Recommended pattern today:** run the blocking call in a thread pool so your ASGI event loop stays responsive:

```python
import asyncio
from functools import partial

async def ask_natural_language(agent, question: str) -> dict:
    loop = asyncio.get_running_loop()
    return await loop.run_in_executor(
        None,
        partial(agent.ask, question),
    )
```

Python 3.9+ alternative:

```python
result = await asyncio.to_thread(agent.ask, question)
```

Reuse a **single** `QueryAgent` instance per process (or use a small pool) so the value index and memory are not rebuilt on every request.

---

## 5. Validate Without a Database

Use `validate_query_plan` to check plans in unit tests, CI, or a plan preview endpoint — no database connection required:

```python
errors = intentql.validate_query_plan(
    query_plan=plan,
    schema_path="config/schema.yaml",
)

if errors:
    for e in errors:
        print(e)
    # "$.filters[0].field: unknown column 'revenue' on table 'orders'"
else:
    print("Plan is valid")
```

---

## Next Steps

| I want to… | Go to |
|---|---|
| Understand joins, rollup, relative dates | [QueryPlan Reference](query-plan-reference.md) |
| Use a free LLM (Gemini, Groq) | [LLM Integration](llm-integration.md) |
| Use IntentQL from FastAPI / async code | [§ Async web apps](#async-apps) (above) |
| Learn the full compilation pipeline | [Core Concepts](concepts.md) |
| See every public function | [API Reference](api-reference.md) |
| Handle errors in production | [Exception Reference](exceptions.md) |
