# IntentQL Analytics Cookbook

This cookbook provides practical examples of common analytical questions, their underlying schema definitions, how IntentQL translates them into deterministic `QueryPlans`, and the resulting parameterized SQL.

---

## 1. Detail & Lookup Queries
**Scenario:** Fetching specific attributes for a single entity or filtering down to micro-level details.

### Question
"Show me the email, registration date, and subscription status for user ID 45021."

### Schema Configuration (`schema.yaml`)
```yaml
tables:
  users:
    description: "Core user accounts and profiles"
    columns:
      user_id: { type: "INTEGER", primary_key: true }
      email: { type: "VARCHAR", description: "User login email address" }
      created_at: { type: "TIMESTAMP", description: "Account registration date" }
      status: { type: "VARCHAR", description: "Subscription status: active, trialing, canceled" }
```

### Extracted QueryPlan
```json
{
  "target_table": "users",
  "operation": "SELECT",
  "select_columns": ["email", "created_at", "status"],
  "filters": [
    {
      "column": "user_id",
      "operator": "EQUALS",
      "value": 45021
    }
  ]
}
```

### Compiled SQL
```sql
SELECT 
    email, 
    created_at, 
    status 
FROM users 
WHERE user_id = :param_1;
```

---

## 2. Trends Over Time
**Scenario:** Time-series aggregations bucketed by intervals (e.g., daily, monthly) to monitor volume patterns.

### Question
"Give me the monthly count of completed orders throughout 2025."

### Schema Configuration (`schema.yaml`)
```yaml
tables:
  orders:
    description: "Customer transactions and order fulfillment records"
    columns:
      order_id: { type: "INTEGER", primary_key: true }
      order_date: { type: "TIMESTAMP", description: "When the order was placed" }
      status: { type: "VARCHAR", description: "Fulfillment state: completed, pending, refunded" }
```

### Extracted QueryPlan
```json
{
  "target_table": "orders",
  "operation": "AGGREGATE",
  "aggregations": [
    { "function": "COUNT", "column": "order_id", "alias": "order_count" }
  ],
  "dimensions": [
    { "column": "order_date", "time_bucket": "MONTH" }
  ],
  "filters": [
    { "column": "status", "operator": "EQUALS", "value": "completed" },
    { "column": "order_date", "operator": "GREATER_THAN_OR_EQUAL", "value": "2025-01-01" },
    { "column": "order_date", "operator": "LESS_THAN_OR_EQUAL", "value": "2025-12-31" }
  ],
  "sort": [{ "column": "order_date", "direction": "ASC" }]
}
```

### Compiled SQL
```sql
SELECT 
    DATE_TRUNC('month', order_date) AS order_date_month, 
    COUNT(order_id) AS order_count 
FROM orders 
WHERE status = :param_1 
  AND order_date >= :param_2 
  AND order_date <= :param_3 
GROUP BY DATE_TRUNC('month', order_date) 
ORDER BY order_date_month ASC;
```

---

## 3. Ratios & Percentages
**Scenario:** Calculating performance or operational metrics by dividing a specific subset by a total dataset.

### Question
"What is the cancellation rate of our items?"

### Schema Configuration (`schema.yaml`)
```yaml
tables:
  line_items:
    description: "Individual product lines tied to transactions"
    columns:
      item_id: { type: "INTEGER", primary_key: true }
      fulfillment_status: { type: "VARCHAR", description: "Status: shipped, processing, cancelled" }
```

### Extracted QueryPlan
```json
{
  "target_table": "line_items",
  "operation": "AGGREGATE",
  "aggregations": [
    {
      "function": "RATIO",
      "numerator_filter": { "column": "fulfillment_status", "operator": "EQUALS", "value": "cancelled" },
      "denominator_column": "item_id",
      "alias": "cancellation_rate"
    }
  ]
}
```

### Compiled SQL
```sql
SELECT 
    COUNT(CASE WHEN fulfillment_status = :param_1 THEN 1 END)::NUMERIC / 
    NULLIF(COUNT(item_id), 0) AS cancellation_rate 
FROM line_items;
```

---

## 4. Ranked Aggregate Queries
**Scenario:** Finding top or bottom performers using groups, sorting, and row limits.

### Question
"Who are the top 5 clients by total spend?"

### Schema Configuration (`schema.yaml`)
```yaml
tables:
  invoices:
    description: "Billing invoices issued to clients"
    columns:
      invoice_id: { type: "INTEGER", primary_key: true }
      client_name: { type: "VARCHAR", description: "The name of the corporate client" }
      amount_usd: { type: "NUMERIC", description: "Gross total invoice amount in USD" }
```

### Extracted QueryPlan
```json
{
  "target_table": "invoices",
  "operation": "AGGREGATE",
  "dimensions": ["client_name"],
  "aggregations": [
    { "function": "SUM", "column": "amount_usd", "alias": "total_spend" }
  ],
  "sort": [
    { "column": "total_spend", "direction": "DESC" }
  ],
  "limit": 5
}
```

### Compiled SQL
```sql
SELECT 
    client_name, 
    SUM(amount_usd) AS total_spend 
FROM invoices 
GROUP BY client_name 
ORDER BY total_spend DESC 
LIMIT 5;
```