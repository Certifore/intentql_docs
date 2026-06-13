<div class="qce-home" markdown>

<div class="qce-hero" markdown>

<div class="qce-hero__eyebrow">Open-source semantic compiler</div>

<h1 class="qce-hero__title">GroundedQL</h1>

<p class="qce-hero__lead">Reliable natural-language analytics without giving an LLM control of your database.</p>

<p class="qce-hero__subtitle">
GroundedQL lets a model provide semantic hints, then uses schema-aware deterministic
infrastructure to resolve those hints into a typed plan, validate it, and compile
parameterized Postgres SQL. The model interprets language. The compiler owns execution.
</p>

<div class="qce-hero__actions">
  <a href="getting-started/" class="qce-btn qce-btn--primary">Get started</a>
  <a href="architecture/" class="qce-btn qce-btn--outline">Why a compiler?</a>
  <a href="https://github.com/Certifore/groundedql" class="qce-btn qce-btn--outline">View source</a>
</div>

<div class="qce-badges">
  <span class="qce-badge qce-badge--blue">Python 3.10+</span>
  <span class="qce-badge qce-badge--green">Postgres</span>
  <span class="qce-badge qce-badge--amber">Model-independent</span>
  <span class="qce-badge qce-badge--gray">SQLAlchemy 2</span>
  <span class="qce-badge qce-badge--green">Open source</span>
</div>

</div>

<div class="qce-pipeline" aria-label="GroundedQL pipeline">
  <div class="qce-pipeline__step">
    <span class="qce-pipeline__num">01</span>
    <strong>Interpret</strong>
    <span>Model proposes semantic hints</span>
  </div>
  <div class="qce-pipeline__arrow">→</div>
  <div class="qce-pipeline__step">
    <span class="qce-pipeline__num">02</span>
    <strong>Resolve</strong>
    <span>Schema and evidence ground meaning</span>
  </div>
  <div class="qce-pipeline__arrow">→</div>
  <div class="qce-pipeline__step">
    <span class="qce-pipeline__num">03</span>
    <strong>Validate</strong>
    <span>Typed plans enforce boundaries</span>
  </div>
  <div class="qce-pipeline__arrow">→</div>
  <div class="qce-pipeline__step">
    <span class="qce-pipeline__num">04</span>
    <strong>Compile</strong>
    <span>Deterministic, parameterized SQL</span>
  </div>
</div>

## Why GroundedQL Exists

Text-to-SQL systems often make a probabilistic model responsible for interpretation,
schema selection, joins, SQL syntax, and safety at the same time. When the answer is wrong,
it is difficult to tell which responsibility failed.

GroundedQL introduces a semantic compiler between the model and the database.

| Raw model-generated SQL | GroundedQL |
|---|---|
| Model produces executable SQL | Model output is treated as untrusted hints |
| Schema policy lives in a prompt | Tables and columns are checked against an allowlist |
| User values may become SQL text | Values are emitted as bind parameters |
| Join behavior depends on generation | Joins must follow declared schema links |
| Model and prompt changes alter everything | Compiler behavior is independently testable |
| Invalid guesses may look plausible | Unknown references fail validation |

GroundedQL does not claim that a compiler makes language unambiguous. It makes the boundary
between uncertain interpretation and database execution explicit, inspectable, and testable.

## Current Evidence

<div class="qce-stats" markdown>
<div class="qce-stat">
<span class="qce-stat__num">50 / 50</span>
<span class="qce-stat__label">Injection-handling property tests</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">20 / 20</span>
<span class="qce-stat__label">Deterministic compilation tests</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">30 / 30</span>
<span class="qce-stat__label">Unknown-schema rejection tests</span>
</div>
<div class="qce-stat">
<span class="qce-stat__num">115 / 130</span>
<span class="qce-stat__label">Partial BIRD Mini-Dev execution matches</span>
</div>
</div>

The BIRD result is a **partial development evaluation** over case IDs 20-149 using provided
evidence. It is not an official leaderboard score. See [Benchmarks](benchmarks.md) for the
methodology, caveats, and publication standard.

## What The Compiler Owns

<div class="qce-features" markdown>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Schema</div>
<p class="qce-feature__title">Allowlisted access</p>
<p class="qce-feature__desc">Only configured tables, columns, and links can enter a valid plan. Unknown references fail before execution.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Resolution</div>
<p class="qce-feature__title">Grounded semantic planning</p>
<p class="qce-feature__desc">Names, types, descriptions, values, keys, dates, and relationships help resolve model hints into valid plan elements.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">IR</div>
<p class="qce-feature__title">Typed QueryPlan contract</p>
<p class="qce-feature__desc">A structured intermediate representation separates language interpretation from executable SQL.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Safety</div>
<p class="qce-feature__title">Parameterized SQL</p>
<p class="qce-feature__desc">Filter values become bind parameters. The model never receives a path for directly executing generated SQL.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Models</div>
<p class="qce-feature__title">Use expensive models less</p>
<p class="qce-feature__desc">The compiler performs structural work so a model can focus on compact semantic hints. Mistral, Ollama, and custom adapters are supported.</p>
</div>
<div class="qce-feature" markdown>
<div class="qce-feature__icon">Evaluation</div>
<p class="qce-feature__title">General improvements only</p>
<p class="qce-feature__desc">Benchmark failures guide reusable compiler capabilities. Domain-specific benchmark branches are not accepted in core logic.</p>
</div>
</div>

## Quick Start

```bash
pip install groundedql
groundedql init --db "postgresql://user:pass@host/db"
```

```python
from sqlalchemy import create_engine
from groundedql.agent import QueryAgent

engine = create_engine("postgresql+psycopg2://user:pass@host/db")

agent = QueryAgent(
    engine=engine,
    schema_path="config/schema.yaml",
    llm="mistral",  # or "ollama", an adapter, or a compatible client
)

result = agent.ask("Which customers placed the most orders last year?")
print(result["rows"])
print(result["sql"])
```

GroundedQL can also validate and execute hand-written `QueryPlan` objects without any model.
See [Getting Started](getting-started.md) for setup and provider configuration.

## Open By Design

The public core includes the semantic plan formats, schema resolver, planner, validator,
compiler, executor, adapters, and benchmark harnesses. You can run it locally against your
own model and database.

GroundedQL is licensed under Apache License 2.0. Contributions are proposed through pull
requests and become part of the official project after review and merge by the lead
maintainer.

[Read the open-source project principles](open-source.md)

## Explore

<div class="qce-path-grid" markdown>
<a class="qce-path-card" href="architecture/">
  <span class="qce-path-card__kicker">Architecture</span>
  <span class="qce-path-card__title">Why a semantic compiler?</span>
  <span class="qce-path-card__desc">Understand the trust boundary between model hints and deterministic execution.</span>
  <span class="qce-path-card__cta">Read architecture →</span>
</a>
<a class="qce-path-card" href="benchmarks/">
  <span class="qce-path-card__kicker">Evidence</span>
  <span class="qce-path-card__title">Benchmarks and limitations</span>
  <span class="qce-path-card__desc">See current results, methodology, caveats, and the standard for future claims.</span>
  <span class="qce-path-card__cta">Inspect evaluation →</span>
</a>
<a class="qce-path-card" href="query-plan-reference/">
  <span class="qce-path-card__kicker">Reference</span>
  <span class="qce-path-card__title">QueryPlan grammar</span>
  <span class="qce-path-card__desc">Explore the typed intermediate representation compiled by GroundedQL.</span>
  <span class="qce-path-card__cta">Open reference →</span>
</a>
<a class="qce-path-card" href="open-source/">
  <span class="qce-path-card__kicker">Community</span>
  <span class="qce-path-card__title">Open-source direction</span>
  <span class="qce-path-card__desc">Learn what belongs in the public core and how to contribute without reward hacking.</span>
  <span class="qce-path-card__cta">Read project principles →</span>
</a>
</div>

</div>
