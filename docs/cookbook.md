# IntentQL Analytics Cookbook

This cookbook provides practical examples of common analytical questions, their underlying schema definitions, how IntentQL translates them into deterministic `QueryPlans`, and the resulting parameterized SQL.

---

## 1. Detail & Lookup Queries
**Scenario:** Fetching specific attributes for a single entity or filtering down to micro-level details.

### Question
"Show me the email, registration date, and subscription status for user ID 45021."

### Schema Configuration (`schema.yaml`)
```yaml
version: 1
dialect: postgres

tables:
  - name: users
    db_table: users
    description: "Core user accounts and profiles"
    primary_id: user_id
    columns:
      - name: user_id
        db_column: user_id
        type: integer
      - name: email
        db_column: email
        type: varchar
        description: "User login email address"
      - name: created_at
        db_column: created_at
        type: timestamp
        description: "Account registration date"
      - name: status
        db_column: status
        type: varchar
        description: "Subscription status: active, trialing, canceled"
        {
  "version": "1.0",
  "dataset": "users",
  "dimensions": [
    {"field": "users.email", "alias": "email"},
    {"field": "users.created_at", "alias": "created_at"},
    {"field": "users.status", "alias": "status"}
  ],
  "metrics": [],
  "filters": [
    {
      "field": "users.user_id",
      "op": "=",
      "value": 45021
    }
  ],
  "order_by": [],
  "limit": 100
}
## 1. Detail & Lookup Queries
**Scenario:** Time-series aggregations bucketed by intervals (e.g., daily, monthly) to monitor volume patterns.

### Question
"Give me the monthly count of completed orders throughout 2025."

### Schema Configuration (`schema.yaml`)
```yaml
version: 1
dialect: postgres

tables:
  - name: orders
    db_table: orders
    description: "Customer transactions and order fulfillment records"
    primary_id: order_id
    primary_date: order_date
    columns:
      - name: order_id
        db_column: order_id
        type: integer
      - name: order_date
        db_column: order_date
        type: timestamp
        description: "When the order was placed"
      - name: status
        db_column: status
        type: varchar
        description: "Fulfillment state: completed, pending, refunded"

        {
  "version": "1.0",
  "dataset": "orders",
  "dimensions": [
    {"field": "orders.order_date", "alias": "order_date_month"}
  ],
  "metrics": [
    {"agg": "count_distinct", "field": "orders.order_id", "alias": "order_count"}
  ],
  "filters": [
    { "field": "orders.status", "op": "=", "value": "completed" },
    { "field": "orders.order_date", "op": ">=", "value": "2025-01-01" },
    { "field": "orders.order_date", "op": "<=", "value": "2025-12-31" }
  ],
  "order_by": [
    { "by": "order_date_month", "dir": "asc" }
  ],
  "limit": 100
}

SELECT 
    DATE_TRUNC('month', order_date) AS order_date_month, 
    COUNT(DISTINCT order_id) AS order_count 
FROM orders 
WHERE status = :param_1 
  AND order_date >= :param_2 
  AND order_date <= :param_3 
GROUP BY DATE_TRUNC('month', order_date) 
ORDER BY order_date_month ASC;