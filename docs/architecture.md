# Why a Semantic Compiler?

Large language models are useful at interpreting language. They are much less dependable
as database runtimes.

IntentQL separates those responsibilities. A model proposes a small amount of semantic
information; deterministic infrastructure resolves that information against an allowlisted
schema, constructs a typed plan, validates it, and compiles parameterized SQL.

```text
Question
   |
   v
Model-generated semantic hints
   |
   v
Schema and evidence resolution
   |
   v
Typed QueryPlan
   |
   v
Validation and semantic lint
   |
   v
Deterministic compiler
   |
   v
Parameterized Postgres SQL
```

The model is an interpreter at the edge of the system. It is not the authority on table
names, join paths, SQL syntax, or execution policy.

## The Design Bet

Most text-to-SQL systems ask a model to produce the final program. IntentQL asks the model
for hints and gives the compiler responsibility for the hard guarantees.

| Concern | Model responsibility | IntentQL responsibility |
|---|---|---|
| Understand the user's language | Propose likely intent and evidence | Normalize and resolve it |
| Choose tables and columns | Suggest semantic targets | Verify against the schema allowlist |
| Build joins | Provide context when useful | Select valid schema links |
| Generate SQL | None | Compile from a typed plan |
| Handle user values | Identify candidate values | Resolve and bind as parameters |
| Reject invalid plans | None | Validate before execution |

This division makes the system less dependent on a specific model. A stronger model can
provide better hints, but it does not receive permission to bypass the compiler.

## The Intermediate Representation

`QueryPlan` is the contract between probabilistic interpretation and deterministic
execution. It describes the requested operation without containing raw SQL.

```json
{
  "dataset": "orders",
  "dimensions": [
    {"field": "ship_country", "alias": "country"}
  ],
  "metrics": [
    {"agg": "count_distinct", "field": "order_id", "alias": "orders"}
  ],
  "order_by": [
    {"by": "orders", "dir": "desc"}
  ],
  "limit": 10
}
```

Because the plan is structured, IntentQL can inspect it before generating SQL:

- every referenced table and column must be allowlisted;
- joins must use declared schema links;
- metric, grouping, and ordering relationships can be validated;
- values become bind parameters rather than SQL text;
- the same valid plan compiles to the same SQL.

## Resolution, Not Memorization

IntentQL is intended to generalize across domains. Core compiler logic must not contain
special cases for benchmark databases, customer names, or known questions.

Instead, the resolver works from reusable signals:

- table and column names;
- descriptions and types;
- primary IDs and primary dates;
- foreign-key links;
- values observed in the connected database;
- linguistic structure such as ranking, ranges, comparison, grouping, and time.

Domain knowledge belongs in the schema configuration and evidence supplied at runtime.
General semantic capabilities belong in the compiler.

## What Is Deterministic?

IntentQL does **not** claim that natural language interpretation is deterministic. Different
models can still propose different hints for an ambiguous question.

It does provide deterministic behavior after a plan is resolved:

- schema validation;
- join-path enforcement;
- plan normalization;
- SQL compilation;
- bind-parameter construction;
- rejection of unknown schema references.

This distinction matters. Deterministic compilation cannot make an ambiguous question
unambiguous, but it can prevent an uncertain interpretation from becoming arbitrary SQL.

## Trust Boundaries

IntentQL's public compiler is designed around several explicit boundaries:

1. **The model is untrusted.** Its output is data to validate, not code to execute.
2. **The schema is an allowlist.** Undeclared tables and columns are unavailable.
3. **Values are parameters.** User-provided values are not interpolated into SQL.
4. **Execution is separate.** Plans can be reviewed or validated without touching a database.
5. **Uncertainty should be visible.** Unsupported or unresolved requests should fail or
   request clarification rather than silently changing meaning.

## Where IntentQL Is Going

The long-term goal is an expressive semantic IR that can represent a broad portion of
analytical SQL while preserving these trust boundaries. That includes richer formulas,
subqueries, temporal reasoning, comparisons, and multi-step questions.

IntentQL is not trying to prove that models are unnecessary. It is trying to make model
quality one component of the system rather than the system's only line of defense.

Read [Core Concepts](concepts.md) for the current implementation details or inspect the
[QueryPlan Reference](query-plan-reference.md) for the executable plan grammar.
