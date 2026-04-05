# IntentQL Documentation

Documentation, benchmarks, and guides for [IntentQL](https://github.com/Certifore/intentql).

## What is IntentQL?

IntentQL is a Python library that turns natural language questions into deterministic, parameterized Postgres SQL. Instead of letting an LLM generate free-form SQL, IntentQL uses a **two-stage architecture**:

1. **LLM extracts a lightweight QueryIntent** — guided by real database values and few-shot examples from past successful queries
2. **Deterministic code builds a QueryPlan** — normalizing the intent, validating values, and compiling to safe, parameterized SQL

This achieves **99% consistency** across rephrasings of the same question, with injection-proof SQL and schema-allowlist enforcement.

## Building the docs locally

```bash
pip install mkdocs-material
mkdocs serve
```

Then open `http://localhost:8000`.

## Contents

- **[Getting Started](docs/getting-started.md)** — Install, configure, first query
- **[Core Concepts](docs/concepts.md)** — Two-stage pipeline, value index, memory, normalization
- **[Schema Reference](docs/schema-reference.md)** — `schema.yaml` format and options
- **[QueryPlan Reference](docs/query-plan-reference.md)** — Plan grammar, operators, rollup, joins
- **[API Reference](docs/api-reference.md)** — All public symbols and their contracts
- **[LLM Integration](docs/llm-integration.md)** — OpenAI, Gemini, Groq, LangChain, Ollama
- **[Exceptions](docs/exceptions.md)** — Error types and handling
- **[Benchmarks](benchmark/)** — Determinism, hallucination, and injection benchmarks
