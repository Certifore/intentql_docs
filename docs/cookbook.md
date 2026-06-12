# IntentQL Analytics Cookbook



This cookbook provides practical examples of common analytical questions, their underlying schema configurations, the generated `QueryPlan` syntax, and the compiled parameterized SQL output.



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

        description: "Unique user account identifier"

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

```



### Extracted QueryPlan

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

  "limit": 100

}

```



### Compiled SQL

```sql

SELECT

    email,

    created_at,

    status

FROM users

WHERE user_id = :p0

LIMIT 100;

```



## 2. Trends Over Time

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

      - name: status

        db_column: status

        type: varchar



```

### Extracted QueryPlan

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

      "op": "<=",

      "value": "2025-12-31T23:59:59"

    }

  ],

  "dimensions": [

    {

      "field": "order_date",

      "alias": "order_month"

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

  "limit": 100

}

```



### Compiled SQL

```sql

SELECT

    DATE_TRUNC('month', order_date) AS order_month,

    COUNT(DISTINCT order_id) AS order_count

FROM orders

WHERE status = :p0

  AND order_date >= :p1

  AND order_date <= :p2

GROUP BY DATE_TRUNC('month', order_date)

ORDER BY order_month ASC

LIMIT 100;

```

## 3. Ratios & Percentages

**Scenario:** Calculating performance or operational metrics by dividing a specific subset by a total dataset.



### Question

"What is the cancellation rate of our items?"



### Schema Configuration (`schema.yaml`)

```yaml

version: 1

dialect: postgres



tables:

  - name: line_items

    db_table: line_items

    description: "Individual product lines tied to transactions"

    primary_id: item_id

    columns:

      - name: item_id

        db_column: item_id

        type: integer

      - name: fulfillment_status

        db_column: fulfillment_status

        type: varchar

```

### Extracted QueryPlan

```json

{

  "version": "1.0",

  "dataset": "line_items",

  "filters": [],

  "dimensions": [],

  "metrics": [

    {

      "agg": "sum",

      "field": "CASE WHEN fulfillment_status = 'cancelled' THEN 1 ELSE 0 END",

      "alias": "cancelled_count"

    },

    {

      "agg": "count_distinct",

      "field": "item_id",

      "alias": "total_items"

    }

  ],

  "rollup": {

    "metrics": [

      {

        "agg": "avg",

        "field": "cancelled_count / NULLIF(total_items, 0)",

        "alias": "cancellation_rate"

      }

    ],

    "limit": 1

  },

  "order_by": []

}

```

### Compiled SQL

```sql

WITH _inner AS (

    SELECT

        SUM(CASE WHEN fulfillment_status = 'cancelled' THEN 1 ELSE 0 END) AS cancelled_count,

        COUNT(DISTINCT item_id) AS total_items

    FROM line_items

)

SELECT

    cancelled_count::NUMERIC / NULLIF(total_items, 0) AS cancellation_rate

FROM _inner

LIMIT 1;

```

## 4. Ranked Aggregate Queries

**Scenario:** Finding top or bottom performers using groups, sorting, and row limits.

### Question

"Who are the top 5 clients by total spend?"



### Schema Configuration (`schema.yaml`)

```yaml

version: 1

dialect: postgres



tables:

  - name: invoices

    db_table: invoices

    description: "Billing invoices issued to clients"

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

### Extracted QueryPlan

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