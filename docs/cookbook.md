# IntentQL Analytics Cookbook

This cookbook shows complete, domain-neutral examples of common analytics questions. Each example includes a valid `schema.yaml`, a hand-written QueryPlan, and the parameterized SQL and bind parameters produced by IntentQL's deterministic compiler.

The examples require neither an LLM nor a live database. QueryPlans can be validated offline with `intentql.validate_query_plan`, and SQL can be produced directly by the compiler.

## 1. Detail And Lookup Queries

### Question

> Show me the email, registration date, and subscription status for user ID 45021.

### Schema Configuration (`schema.yaml`)

```yaml
tables:
  - name: users
    db_table: users
    primary_id: user_id
    columns:
      - name: user_id
        db_column: user_id
        type: integer
      - name: email
        db_column: email
        type: varchar
      - name: created_at
        db_column: created_at
        type: timestamp
      - name: status
        db_column: status
        type: varchar
```

### QueryPlan

```json
{
  "version": "1.0",
  "dataset": "users",
  "filters": [
    {
      "field": "user_id",
      "op": "=",
      "value": 45021
    }
  ],
  "dimensions": [
    {"field": "email"},
    {"field": "created_at"},
    {"field": "status"}
  ],
  "metrics": [],
  "order_by": [],
  "limit": 100,
  "offset": 0
}
```

### Compiled SQL

```sql
SELECT users_1.created_at, users_1.email, users_1.status
FROM users AS users_1
WHERE users_1.user_id = %(v_1)s
GROUP BY users_1.created_at, users_1.email, users_1.status
LIMIT %(param_1)s OFFSET %(param_2)s
```

```json
{
  "v_1": 45021,
  "param_1": 100,
  "param_2": 0
}
```

## 2. Trends Over Time

### Question

> Give me the monthly count of completed orders throughout 2025.

### Schema Configuration (`schema.yaml`)

```yaml
tables:
  - name: orders
    db_table: orders
    primary_id: order_id
    primary_date: order_date
    columns:
      - name: order_id
        db_column: order_id
        type: integer
      - name: order_date
        db_column: order_date
        type: timestamp
      - name: status
        db_column: status
        type: varchar
```

### QueryPlan

```json
{
  "version": "1.0",
  "dataset": "orders",
  "filters": [
    {
      "field": "status",
      "op": "=",
      "value": "completed"
    },
    {
      "field": "order_date",
      "op": ">=",
      "value": "2025-01-01T00:00:00"
    },
    {
      "field": "order_date",
      "op": "<",
      "value": "2026-01-01T00:00:00"
    }
  ],
  "dimensions": [
    {
      "field": "order_date",
      "alias": "order_month",
      "time_bucket": "month"
    }
  ],
  "metrics": [
    {
      "agg": "count_distinct",
      "field": "order_id",
      "alias": "order_count"
    }
  ],
  "order_by": [
    {
      "by": "order_month",
      "dir": "asc"
    }
  ],
  "limit": 100,
  "offset": 0
}
```

### Compiled SQL

```sql
SELECT date_trunc(%(v_1)s, orders_1.order_date) AS order_month,
       count(DISTINCT orders_1.order_id) AS order_count
FROM orders AS orders_1
WHERE orders_1.order_date < %(v_2)s
  AND orders_1.order_date >= %(v_3)s
  AND orders_1.status = %(v_4)s
GROUP BY date_trunc(%(v_5)s, orders_1.order_date)
ORDER BY order_month ASC NULLS LAST
LIMIT %(param_1)s OFFSET %(param_2)s
```

```json
{
  "v_1": "month",
  "v_2": "2026-01-01T00:00:00",
  "v_3": "2025-01-01T00:00:00",
  "v_4": "completed",
  "v_5": "month",
  "param_1": 100,
  "param_2": 0
}
```

## 3. Ratios And Percentages

### Question

> What is the cancellation rate of our items?

### Schema Configuration (`schema.yaml`)

```yaml
tables:
  - name: line_items
    db_table: line_items
    primary_id: item_id
    columns:
      - name: item_id
        db_column: item_id
        type: integer
      - name: fulfillment_status
        db_column: fulfillment_status
        type: varchar
```

### QueryPlan

This advanced expression counts cancelled items and divides by the total distinct item count. `nullif` prevents division by zero.

```json
{
  "version": "1.0",
  "dataset": "line_items",
  "select": [
    {
      "expr": {
        "op": "/",
        "args": [
          {
            "op": "*",
            "args": [
              {
                "func": "sum",
                "args": [
                  {
                    "case": {
                      "whens": [
                        {
                          "when": {
                            "cmp": {
                              "left": {"col": "fulfillment_status"},
                              "op": "=",
                              "right": "cancelled"
                            }
                          },
                          "then": {"lit": 1}
                        }
                      ],
                      "else": {"lit": 0}
                    }
                  }
                ]
              },
              {"lit": 1.0}
            ]
          },
          {
            "func": "nullif",
            "args": [
              {
                "func": "count_distinct",
                "args": [{"col": "item_id"}]
              },
              {"lit": 0}
            ]
          }
        ]
      },
      "alias": "cancellation_rate"
    }
  ],
  "limit": 1,
  "offset": 0
}
```

### Compiled SQL

```sql
SELECT (sum(CASE
              WHEN (line_items_1.fulfillment_status = %(v_1)s) THEN %(v_2)s
              ELSE %(v_3)s
            END) * %(v_4)s)
       / nullif(count(DISTINCT line_items_1.item_id), %(v_5)s) AS cancellation_rate
FROM line_items AS line_items_1
LIMIT %(param_1)s OFFSET %(param_2)s
```

```json
{
  "v_1": "cancelled",
  "v_2": 1,
  "v_3": 0,
  "v_4": 1.0,
  "v_5": 0,
  "param_1": 1,
  "param_2": 0
}
```

## 4. Ranked Aggregate Queries

### Question

> Who are the top 5 clients by total spend?

### Schema Configuration (`schema.yaml`)

```yaml
tables:
  - name: invoices
    db_table: invoices
    primary_id: invoice_id
    columns:
      - name: invoice_id
        db_column: invoice_id
        type: integer
      - name: client_name
        db_column: client_name
        type: varchar
      - name: amount_usd
        db_column: amount_usd
        type: numeric
```

### QueryPlan

```json
{
  "version": "1.0",
  "dataset": "invoices",
  "filters": [],
  "dimensions": [
    {"field": "client_name"}
  ],
  "metrics": [
    {
      "agg": "sum",
      "field": "amount_usd",
      "alias": "total_spend"
    }
  ],
  "order_by": [
    {
      "by": "total_spend",
      "dir": "desc"
    }
  ],
  "limit": 5,
  "offset": 0
}
```

### Compiled SQL

```sql
SELECT invoices_1.client_name, sum(invoices_1.amount_usd) AS total_spend
FROM invoices AS invoices_1
GROUP BY invoices_1.client_name
ORDER BY total_spend DESC NULLS LAST
LIMIT %(param_1)s OFFSET %(param_2)s
```

```json
{
  "param_1": 5,
  "param_2": 0
}
```
