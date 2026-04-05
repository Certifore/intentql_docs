# Core Concepts

Understanding how IntentQL works end-to-end will help you configure it correctly and debug issues when they arise.

<div class="qce-path-grid" markdown>
<a class="qce-path-card" href="getting-started/">
  <span class="qce-path-card__kicker">Step 1</span>
  <span class="qce-path-card__title">Getting Started</span>
  <span class="qce-path-card__desc">Install IntentQL, define schema allowlists, and run your first execution path.</span>
  <span class="qce-path-card__cta">Open guide -></span>
</a>
<a class="qce-path-card" href="query-plan-reference/">
  <span class="qce-path-card__kicker">Step 3</span>
  <span class="qce-path-card__title">QueryPlan Reference</span>
  <span class="qce-path-card__desc">Use the full plan grammar, operators, rollup, joins, and advanced expressions.</span>
  <span class="qce-path-card__cta">Read reference -></span>
</a>
<a class="qce-path-card" href="api-reference/">
  <span class="qce-path-card__kicker">Production</span>
  <span class="qce-path-card__title">API Contracts</span>
  <span class="qce-path-card__desc">Integrate planner/compiler/executor entry points with typed error handling.</span>
  <span class="qce-path-card__cta">Open API docs -></span>
</a>
</div>

!!! abstract "TL;DR"
    IntentQL separates **intent** (what data you want) from **execution** (how to fetch it). The LLM extracts a lightweight `QueryIntent`; deterministic code normalizes it, validates it against real database values, and builds a `QueryPlan`; the compiler turns that plan into parameterized SQL. The layers never mix.

---

## The Two-Stage Pipeline

IntentQL uses a **two-stage architecture** that minimizes LLM responsibility and maximizes determinism:

```
Natural language question
        │
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  1. Few-Shot Memory Lookup                          │
 │     IntentMemory.retrieve()                         │
 │                                                     │
 │  • Embeds the question (OpenAI text-embedding)      │
 │  • Finds similar past questions in ChromaDB         │
 │  • Returns verified (question, intent) examples     │
 └─────────────────────────────────────────────────────┘
        │  few-shot examples
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  2. LLM Intent Extraction                           │
 │     extract_intent()                                │
 │                                                     │
 │  • LLM receives: schema + value index pick-lists    │
 │    + few-shot examples + question                   │
 │  • LLM produces: QueryIntent JSON                   │
 │    (dataset, keyword, filters, time_range,           │
 │     aggregation, group_by, sort, limit)              │
 └─────────────────────────────────────────────────────┘
        │  raw intent
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  3. Deterministic Normalization                     │
 │     normalize_intent()                              │
 │                                                     │
 │  • Absorbs redundant keyword/filter overlap         │
 │  • Ensures group_by is always a list                │
 │  • Auto-adds group_by for multi-value filters       │
 └─────────────────────────────────────────────────────┘
        │  normalized intent
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  4. Value Index Validation + Retry                  │
 │     validate_intent_against_index()                 │
 │                                                     │
 │  • Checks filter values exist in the database       │
 │  • If issues found → feeds feedback to LLM → retry  │
 │  • Fuzzy-resolves close matches automatically       │
 └─────────────────────────────────────────────────────┘
        │  validated intent
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  5. Deterministic Plan Builder                      │
 │     build_plan_from_intent()                        │
 │                                                     │
 │  • Intent + schema metadata → QueryPlan JSON        │
 │  • Keyword → OR across keyword_search_or columns   │
 │  • Time range → $relative_date sentinels            │
 │  • count → count_distinct(primary_id)               │
 │  • Auto-excludes null/empty on group-by columns     │
 └─────────────────────────────────────────────────────┘
        │  QueryPlan dict
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  6. Compiler + Executor                             │
 │     Compiler.compile() → Executor.execute()         │
 │                                                     │
 │  • Validates against schema.yaml allowlist           │
 │  • All values → bind parameters                     │
 │  • Auto-injects JOIN paths via BFS                  │
 │  • Executes with statement timeout                  │
 └─────────────────────────────────────────────────────┘
        │
        ▼
 ┌─────────────────────────────────────────────────────┐
 │  7. Memory Storage                                  │
 │     IntentMemory.store()                            │
 │                                                     │
 │  • Stores (question, normalized_intent) in ChromaDB │
 │  • Future similar questions get this as few-shot    │
 └─────────────────────────────────────────────────────┘
        │
        ▼
   {"rows": [...], "row_count": N, "columns": [...], "sql": "..."}
```

The key insight: **the LLM only decides _what_ the user wants (intent), never _how_ to query it (SQL)**. Every structural decision — keyword OR clauses, date range filters, count vs. count_distinct, join paths — is made by deterministic code using schema metadata.

---

## Why Two Stages?

The original IntentQL architecture asked the LLM to produce a full `QueryPlan` directly. This worked for simple queries but had a fundamental consistency problem: the same question phrased differently could produce structurally different plans, leading to different SQL and different results.

The two-stage architecture solves this by:

1. **Reducing the LLM's task** — extracting a flat intent (7-8 fields) is far simpler and more constrained than producing a nested QueryPlan.
2. **Normalizing after extraction** — deterministic code canonicalizes the intent so equivalent phrasings produce identical structures.
3. **Learning from success** — the memory stores verified intents and injects them as few-shot examples, teaching the LLM to be consistent without manual training.
4. **Validating against reality** — the value index catches hallucinated or misspelled values before they reach SQL.

---

## The Value Index

At startup, `QueryAgent` queries the database for distinct values of categorical columns (building names, status codes, asset types, etc.) and caches them in memory.

```python
value_index = build_value_index(engine, schema_path)
# {
#   "work_orders": {
#     "building_name": ["ATHENAEUM", "BECHTEL RESIDENCE", ...],
#     "status_code": ["CLOSED", "COMPLETE", "IN PROGRESS", ...],
#   },
#   "assets": {
#     "keyword_of_asset": ["AIR HANDLER", "FIRE EXTINGUISHER", ...],
#   },
# }
```

The value index serves three purposes:

1. **Prompt injection** — real values are formatted as pick-lists and injected into the LLM prompt, so the LLM picks from actual database values instead of guessing.
2. **Post-extraction validation** — after the LLM extracts an intent, filter values are checked against the index. Unknown values trigger a retry with feedback.
3. **Fuzzy resolution** — close matches (e.g., "bechtel" → "BECHTEL RESIDENCE") are resolved automatically using substring matching and difflib.

The index is built once at server startup and shared across all user sessions. It only rebuilds on server restart, which naturally picks up data changes.

!!! tip "Which columns get indexed?"
    IntentQL indexes `varchar`/`text` columns that are likely categorical — building names, status codes, priority codes, asset keywords. It skips free-text columns (descriptions, comments) and high-cardinality ID columns. See `value_index._get_indexable_columns()` for the heuristic.

---

## Intent Normalization

After the LLM extracts an intent, deterministic normalization rules are applied to resolve structural ambiguities:

| Rule | What it does |
|---|---|
| **Absorb keyword filters** | If the LLM put a keyword as both `keyword` and a filter on a `keyword_search_or` column, the redundant filter is removed. The keyword generates a broad OR search that subsumes the narrower filter. |
| **Ensure group_by is a list** | Normalizes `group_by` from string or null to always be a list. |
| **Auto group_by for multi-value filters** | If a filter has multiple values (e.g., buildings A, B, C) and the aggregation is count/sum/avg, the filtered column is added to `group_by` so results are broken down per value. |

These rules ensure that no matter how the LLM structures the intent, the normalized output is identical for semantically equivalent questions.

---

## Few-Shot Memory (IntentMemory)

The `IntentMemory` is IntentQL's learning mechanism. Every time a query succeeds, the (question, normalized_intent) pair is stored in a ChromaDB vector database. When a new question comes in:

1. The question is embedded using `text-embedding-3-small`.
2. ChromaDB finds the most similar past questions (cosine similarity).
3. Matching examples (above a similarity threshold) are formatted and injected into the LLM prompt.

This means the LLM sees concrete examples of "for this similar question, here's the intent that worked." Over time, the system becomes more consistent as it accumulates verified examples.

```
New question: "how many electrical issues in bechtel last year?"
    │
    ▼  retrieve similar
Similar past question: "how many plumbing issues in bechtel last year?" (sim: 0.92)
    │
    ▼  inject as few-shot
LLM sees the working intent and follows the same structure
```

!!! info "Memory persistence"
    Memory persists across server restarts via ChromaDB's on-disk storage. It is shared across all users of the same deployment — one user's successful queries improve consistency for everyone.

---

## The Schema: Your Allowlist

`schema.yaml` is the **single source of truth** for everything the compiler is allowed to touch.

!!! danger "Security guarantee"
    If a table or column is not declared in `schema.yaml`, it **cannot** appear in compiled SQL — not from the LLM, not from a hand-written plan, not from user input of any kind. The compiler raises `QueryPlanError` and execution never happens.

The schema defines:

=== "Tables & Columns"

    ```yaml
    tables:
      - name: orders               # logical name — the LLM uses this
        db_table: '"Orders"'       # physical name — the DB sees this
        description: "Customer orders"
        primary_id: order_id       # used for count_distinct
        primary_date: order_date   # used for time range filters
        columns:
          - name: order_id
            db_column: '"OrderID"'
            type: integer
          - name: freight
            db_column: '"Freight"'
            type: numeric
    ```

=== "Links (Joins)"

    ```yaml
    links:
      - name: orders_to_customers
        from_table: orders
        to_table: customers
        join_type: left
        "on":
          - left: orders.customer_id
            op: "="
            right: customers.customer_id
    ```

=== "Keyword Search"

    ```yaml
    tables:
      - name: work_orders
        keyword_search_or:
          - work_order_description
          - long_desc
          - shop_name
    ```

    When `keyword_search_or` is set, keyword queries generate an OR clause across all listed columns — e.g., `WHERE desc ILIKE '%plumbing%' OR long_desc ILIKE '%plumbing%'`.

---

## The QueryIntent: Lightweight Structured Intent

A `QueryIntent` captures _what the user wants_ in a flat, simple structure:

```json
{
  "dataset": "work_orders",
  "keyword": "plumbing",
  "filters": [
    {"column": "building_name", "values": ["BECHTEL RESIDENCE", "PAGE HOUSE"]}
  ],
  "time_range": "last_year",
  "aggregation": "count",
  "group_by": ["building_name"],
  "sort_direction": "desc",
  "limit": null
}
```

The LLM only needs to fill in these fields — it never decides on SQL operators, join paths, date formats, or count strategy. Those are all handled by the deterministic plan builder.

---

## The QueryPlan: Structured Execution Plan

The plan builder converts a `QueryIntent` into a `QueryPlan`:

```json
{
  "version": "1.0",
  "dataset": "work_orders",
  "filters": [
    {"field": "building_name", "op": "in", "value": ["BECHTEL RESIDENCE", "PAGE HOUSE"]},
    {"field": "created_date", "op": ">=", "value": "2025-01-01T00:00:00"},
    {"field": "created_date", "op": "<", "value": "2026-01-01T00:00:00"},
    {"field": "building_name", "op": "!=", "value": ""},
    {"field": "building_name", "op": "is_not_null", "value": true}
  ],
  "dimensions": [{"field": "building_name", "alias": "building_name"}],
  "metrics": [{"agg": "count_distinct", "field": "work_order_id", "alias": "total"}],
  "where": {
    "or": [
      {"cmp": {"left": {"col": "work_order_description"}, "op": "contains", "right": "plumbing"}},
      {"cmp": {"left": {"col": "long_desc"}, "op": "contains", "right": "plumbing"}}
    ]
  },
  "order_by": [{"by": "total", "dir": "desc"}],
  "limit": 100
}
```

The compiler then turns this into parameterized SQL:

```sql
SELECT "building_name", COUNT(DISTINCT "workOrderId") AS total
FROM "finalWorkOrders"
WHERE "building_name" IN (%(p0)s, %(p1)s)
  AND "createdDate" >= %(p2)s AND "createdDate" < %(p3)s
  AND "building_name" != %(p4)s AND "building_name" IS NOT NULL
  AND ("workOrderDescription" ILIKE %(p5)s OR "longDesc" ILIKE %(p6)s)
GROUP BY "building_name"
ORDER BY total DESC
LIMIT 100
```

!!! note "No SQL in the plan"
    The plan has no `FROM`, `WHERE`, `GROUP BY`, or SQL keywords — those are the compiler's responsibility. The LLM only declares intent.

See the [QueryPlan Reference](query-plan-reference.md) for the full spec.

---

## Determinism

The compiler is a **pure function**: identical input always produces identical output.

- No randomness in compilation (LLM temperature is isolated to the intent extraction stage)
- Bind params (`p0`, `p1`, ...) are generated from a reset counter on each call
- SHA-256 `plan_hash` attached to every generated plan
- Intent normalization ensures equivalent phrasings produce identical plan structures

---

## Parameterized Values

Every filter value becomes a **named bind parameter** — it is never string-interpolated into SQL:

```python
plan = {"dataset": "orders", "filters": [{"field": "freight", "op": ">", "value": 100}]}
sql, params = Compiler(schema).compile(plan)
# sql:    "SELECT ... FROM orders WHERE freight > %(p0)s"
# params: {"p0": 100}
```

This makes SQL injection structurally impossible regardless of what the LLM (or a user) provides.

---

## Rollup: Two-Level Aggregation

Some questions require aggregating an already-aggregated result — for example: *"What is the average order value per customer?"*

This needs two layers:

1. **Inner query** — sum order value per customer
2. **Outer query** — average those sums

IntentQL models this with `rollup`:

```json
{
  "dataset": "order_details",
  "dimensions": [{"field": "order_details.order_id"}],
  "metrics": [{"agg": "sum", "field": "order_details.unit_price", "alias": "revenue"}],
  "rollup": {
    "metrics": [{"agg": "avg", "field": "revenue", "alias": "avg_revenue"}],
    "limit": 1
  }
}
```

The compiler wraps the inner query in a CTE and applies the outer aggregation over it.

---

## Auto Join Injection

When a plan references columns from multiple tables, IntentQL automatically computes and injects the join path using BFS over the `links` graph in `schema.yaml`.

!!! tip "Explicit joins override auto-injection"
    If you declare joins in the plan's `joins` array, auto-injection is skipped for that plan.

---

## The Spec Builder

The spec builder auto-generates the LLM's system prompt from your `schema.yaml`:

```bash
python -m intentql.spec_builder \
  --schema config/schema.yaml \
  --output config/queryplan_spec_generated.yaml
```

!!! warning "Regenerate after every schema change"
    The spec is the LLM's only knowledge of your database structure. If you add a table and don't regenerate, the LLM cannot reference it.

---

## Legacy Pipeline

IntentQL retains the original "LLM generates full QueryPlan" pipeline as a fallback. If the intent pipeline fails for any reason, `QueryAgent` automatically falls back to the legacy pipeline with retry + autofix.

You can also force the legacy pipeline:

```python
agent = intentql.QueryAgent(
    engine=engine,
    schema_path="config/schema.yaml",
    spec_path="config/queryplan_spec_generated.yaml",
    llm=your_llm,
    use_intent_pipeline=False,  # force legacy
)
```
