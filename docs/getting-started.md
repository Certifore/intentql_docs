# Getting Started

Get from zero to a running IntentQL query in under five minutes.

---

## Prerequisites

- Python ≥ 3.10
- A running Postgres database (local or remote)
- Optional: an LLM API key (OpenAI, Gemini, Groq, etc.) — only needed if you want natural language input
- Optional: `chromadb` — for persistent few-shot memory (recommended for production)

---

## 1. Install

```bash
pip install intentql
```

Install with LLM extras if you plan to use natural language input:

```bash
pip install "intentql[openai]"                   # OpenAI SDK
pip install chromadb                        # few-shot memory (recommended)
pip install langchain-openai                # LangChain + OpenAI
pip install langchain-google-genai          # Gemini (free tier available)
pip install langchain-groq                  # Groq (free, fast)
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
export OPENAI_API_KEY=sk-...
intentql describe --schema config/schema.yaml --db "postgresql://user:pass@host/db"
```

This sends each table's structure and sample values to the LLM, which generates concise descriptions for every table and column. The `--db` flag is optional but recommended — sample values give the LLM much better context (e.g., it can detect "values are in UPPER CASE").

!!! tip "Review and refine"
    The generated descriptions are a great starting point. For best results, review them and add domain-specific guidance. For example, you might add: "Do not filter on this column for trade keywords; use `description` instead."

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

## 4. Validate Without a Database

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
| Learn the full compilation pipeline | [Core Concepts](concepts.md) |
| See every public function | [API Reference](api-reference.md) |
| Handle errors in production | [Exception Reference](exceptions.md) |
