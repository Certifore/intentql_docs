<div class="qce-home" markdown>

<div class="qce-hero" markdown>

<div class="qce-hero__eyebrow">IntentQL</div>

<h1 class="qce-hero__title">LLM-powered queries.<br>Deterministic SQL. Zero trust.</h1>

<p class="qce-hero__subtitle">
IntentQL extracts a lightweight <strong>QueryIntent</strong> from natural language, normalizes it
deterministically, validates it against real database values, and compiles it into
parameterized Postgres SQL — enforcing your schema allowlist, neutralizing injections,
and delivering <strong>consistent results</strong> across rephrasings.
Let your LLM own <em>intent</em>. Let IntentQL own <em>everything else</em>.
</p>

<div class="qce-hero__actions">
  <a href="getting-started/" class="qce-btn qce-btn--primary">→ Get Started</a>
  <a href="concepts/" class="qce-btn qce-btn--outline">How it works</a>
  <a href="https://github.com/Certifore/intentql" class="qce-btn qce-btn--outline">GitHub</a>
</div>

<div class="qce-badges">
  <span class="qce-badge qce-badge--purple">Python ≥ 3.10</span>
  <span class="qce-badge qce-badge--blue">Postgres</span>
  <span class="qce-badge qce-badge--green">MIT License</span>
  <span class="qce-badge qce-badge--gray">SQLAlchemy 2</span>
  <span class="qce-badge qce-badge--green">v0.2.0</span>
</div>

</div>

<div class="qce-stats" markdown>
<div class="qce-stat">
<span class="qce-stat__num">99% Consistent</span>
<span class="qce-stat__label">Same question, same answer</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">Deterministic</span>
<span class="qce-stat__label">Same intent, same SQL</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">Schema-Scoped</span>
<span class="qce-stat__label">Allowlist only access</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">Self-Improving</span>
<span class="qce-stat__label">Learns from successful queries</span>
</div>
</div>

---

## The Problem with LLM-Generated SQL

Every production AI data feature eventually hits the same walls:

<div class="qce-comparison" markdown>

|  | Raw LLM SQL | **IntentQL** |
|---|:---:|:---:|
| SQL injection via prompt | <span class="cross">No: never safe</span> | <span class="check">Yes: bind params always</span> |
| Hallucinated table names | <span class="cross">No: silent wrong answer risk</span> | <span class="check">Yes: hard error, allowlist enforced</span> |
| Non-deterministic output | <span class="cross">No: varies per call</span> | <span class="check">Yes: intent normalization + memory</span> |
| Inconsistent across rephrasings | <span class="cross">No: different SQL each time</span> | <span class="check">Yes: few-shot memory + normalization</span> |
| Wrong filter values | <span class="cross">No: LLM guesses names</span> | <span class="check">Yes: value index from real data</span> |
| LLM picks JOIN strategy | <span class="cross">No: unpredictable</span> | <span class="check">Yes: BFS shortest path, always</span> |
| Needs DB introspection | <span class="cross">No: exposes full schema</span> | <span class="check">Yes: LLM sees logical names only</span> |

</div>

IntentQL solves this with a **two-stage architecture**. The LLM extracts a lightweight `QueryIntent` — no SQL, no table names, no structural decisions. Deterministic code normalizes the intent, validates it against real database values, and builds a correct `QueryPlan` that compiles into safe, parameterized SQL.

---

## How It Works

```
  User question
       │
       ▼
  ┌────────────────────────────────────────────────────┐
  │  Few-Shot Memory (ChromaDB)                        │
  │  Retrieves similar past questions + verified       │  ← learns from success
  │  intents as examples for the LLM                   │
  └────────────────────────────────────────────────────┘
       │
       ▼
  ┌────────────────────────────────────────────────────┐
  │  LLM  (OpenAI · Gemini · Groq · any model)        │
  │  Extracts: QueryIntent JSON                        │  ← lightweight intent, not SQL
  │  Guided by: schema + value index + few-shot        │
  └────────────────────────────────────────────────────┘
       │
       ▼  { "dataset": "orders", "keyword": "...", "filters": [...] }
  ┌────────────────────────────────────────────────────┐
  │  Intent Normalization + Value Validation           │
  │  · Absorbs redundant keyword/filter overlap        │
  │  · Fuzzy-resolves values against real DB data      │  ← deterministic cleanup
  │  · Retries LLM if values don't match              │
  └────────────────────────────────────────────────────┘
       │
       ▼
  ┌────────────────────────────────────────────────────┐
  │  Deterministic Plan Builder                        │
  │  · Intent + schema metadata → QueryPlan JSON       │
  │  · Keyword → OR clause, time → date sentinels     │  ← zero LLM decisions
  │  · count → count_distinct(primary_id)              │
  └────────────────────────────────────────────────────┘
       │
       ▼
  ┌────────────────────────────────────────────────────┐
  │  IntentQL Compiler                                      │
  │  · Validates against schema.yaml allowlist          │
  │  · All values → bind parameters                    │
  │  · Auto-injects JOIN paths via BFS                 │
  └────────────────────────────────────────────────────┘
       │
       ▼  SELECT count(DISTINCT "id") FROM ... WHERE ...
  ┌────────────────────────────────────────────────────┐
  │  Postgres                                          │
  └────────────────────────────────────────────────────┘
       │
       ▼  { "rows": [...], "row_count": 42, "sql": "...", "params": {...} }
```

---

## Quick Start

=== "From zero (CLI)"

    ```bash
    # 1. Generate schema from your database
    intentql init --db "postgresql://user:pass@host/db"

    # 2. Enrich with LLM descriptions (optional, recommended)
    # Works with any OpenAI-compatible provider (OpenAI, Groq, Ollama, etc.)
    export LLM_API_KEY=sk-...
    intentql describe --schema config/schema.yaml --db "postgresql://user:pass@host/db"
    ```

    ```python
    # 3. Ask questions
    from sqlalchemy import create_engine
    from openai import OpenAI
    import intentql

    engine = create_engine("postgresql+psycopg2://user:pass@host/db")

    agent = intentql.QueryAgent(
        engine=engine,
        schema_path="config/schema.yaml",
        llm=OpenAI(api_key="sk-..."),
    )

    result = agent.ask("Top 10 customers by total order value last 90 days")
    print(result["rows"])
    print(result["sql"])
    ```

=== "Hand-written plan"

    ```python
    from sqlalchemy import create_engine
    import intentql

    engine = create_engine("postgresql+psycopg2://user:pass@host/db")

    result = intentql.execute_query_plan(
        engine=engine,
        schema_path="config/schema.yaml",
        query_plan={
            "dataset": "orders",
            "filters": [
                {"field": "ship_country", "op": "=", "value": "Germany"},
                {
                    "field": "order_date",
                    "op": ">=",
                    "value": {"$relative_date": {"op": "now_minus_days", "days": 30}},
                },
            ],
            "metrics": [{"agg": "sum", "field": "freight", "alias": "total_freight"}],
            "limit": 1,
        },
    )
    print(result["rows"])
    print(result["sql"])
    ```

=== "Validate offline"

    ```python
    import intentql

    errors = intentql.validate_query_plan(
        query_plan=plan,
        schema_path="config/schema.yaml",
    )

    if errors:
        for e in errors:
            print(e)
    else:
        print("Plan is valid")
    ```

---

## Feature Highlights

<div class="qce-features" markdown>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Consistency</div>
<p class="qce-feature__title">99% Consistent Across Rephrasings</p>
<p class="qce-feature__desc">Intent normalization + few-shot memory ensure the same question phrased differently produces the same result. The system learns from every successful query.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Accuracy</div>
<p class="qce-feature__title">Value Index Grounding</p>
<p class="qce-feature__desc">At startup, IntentQL indexes real database values and injects them as pick-lists into the LLM prompt. The LLM picks from actual data — no more hallucinated filter values.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Security</div>
<p class="qce-feature__title">Injection-Proof by Design</p>
<p class="qce-feature__desc">Every filter value becomes a named bind parameter. There is no code path that interpolates user input into SQL text — ever.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Schema</div>
<p class="qce-feature__title">Schema Allowlist Enforcement</p>
<p class="qce-feature__desc">Only tables and columns declared in <code>schema.yaml</code> are reachable. Unknown names raise a hard <code>QueryPlanError</code> — no silent degradation.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Memory</div>
<p class="qce-feature__title">Self-Improving via ChromaDB</p>
<p class="qce-feature__desc">Successful (question, intent) pairs are stored in ChromaDB and retrieved as few-shot examples for similar future questions. No manual training required.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">LLM</div>
<p class="qce-feature__title">LLM-Agnostic</p>
<p class="qce-feature__desc">One factory function — <code>make_llm_client()</code> — adapts OpenAI, LangChain, Gemini, Groq, Ollama, or any callable automatically.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">CLI</div>
<p class="qce-feature__title">Zero-Config Setup</p>
<p class="qce-feature__desc">Run <code>intentql init --db URL</code> to auto-generate your schema from the database, then <code>intentql describe</code> to add LLM-powered column descriptions. No manual YAML needed.</p>
</div>
</div>

---

## Install

```bash
pip install intentql
```

```bash
# Optional: few-shot memory (ChromaDB)
pip install "intentql[memory]"
# Your LLM SDK (example — use any provider you wire into QueryAgent)
pip install openai
```

!!! tip "Installing from source"
    ```bash
    git clone https://github.com/Certifore/intentql
    cd intentql
    pip install -e ".[dev]"
    ```

---

## Next Steps

<div class="qce-path-grid" markdown>
<a class="qce-path-card" href="getting-started/">
  <span class="qce-path-card__kicker">Quickstart</span>
  <span class="qce-path-card__title">Get Started in 5 Minutes</span>
  <span class="qce-path-card__desc">Install IntentQL, define your schema, and run your first query against a live database.</span>
  <span class="qce-path-card__cta">Read guide →</span>
</a>
<a class="qce-path-card" href="concepts/">
  <span class="qce-path-card__kicker">Architecture</span>
  <span class="qce-path-card__title">Understand the Pipeline</span>
  <span class="qce-path-card__desc">See how intents are extracted, normalized, validated, and compiled into safe SQL.</span>
  <span class="qce-path-card__cta">Read concepts →</span>
</a>
<a class="qce-path-card" href="query-plan-reference/">
  <span class="qce-path-card__kicker">Reference</span>
  <span class="qce-path-card__title">QueryPlan Format</span>
  <span class="qce-path-card__desc">Operators, aggregations, rollup, set operations, and advanced plan constructs.</span>
  <span class="qce-path-card__cta">Open reference →</span>
</a>
<a class="qce-path-card" href="llm-integration/">
  <span class="qce-path-card__kicker">Integrations</span>
  <span class="qce-path-card__title">Connect Your LLM</span>
  <span class="qce-path-card__desc">Use OpenAI, Gemini, Groq, LangChain, Ollama, or any callable adapter.</span>
  <span class="qce-path-card__cta">Open integration guide →</span>
</a>
</div>

</div>
