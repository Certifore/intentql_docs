# IntentQL Documentation

Documentation, benchmarks, and guides for [IntentQL](https://github.com/Certifore/intentql).

## What is IntentQL?

IntentQL is an open-source semantic compiler for natural-language analytics over Postgres.
A model proposes semantic hints; deterministic infrastructure resolves those hints against
an allowlisted schema, builds a typed QueryPlan, validates it, and compiles parameterized
SQL.

The model interprets language. The compiler owns execution.

## Building the docs locally

```bash
pip install mkdocs-material
mkdocs serve
```

Then open `http://localhost:8000`.

## Contents

- **[Getting Started](docs/getting-started.md)** — Install, configure, first query
- **[Core Concepts](docs/concepts.md)** — Two-stage pipeline, value index, memory, normalization
- **[Why a Compiler?](docs/architecture.md)** — Architecture and trust boundaries
- **[Benchmarks](docs/benchmarks.md)** — Current results, methodology, and limitations
- **[Open Source](docs/open-source.md)** — License, governance, and contribution philosophy
- **[Schema Reference](docs/schema-reference.md)** — `schema.yaml` format and options
- **[QueryPlan Reference](docs/query-plan-reference.md)** — Plan grammar, operators, rollup, joins
- **[API Reference](docs/api-reference.md)** — All public symbols and their contracts
- **[LLM Integration](docs/llm-integration.md)** — OpenAI, Gemini, Groq, LangChain, Ollama
- **[Exceptions](docs/exceptions.md)** — Error types and handling
- **[Benchmark harnesses](benchmark/)** — Determinism, hallucination, and injection property tests

## Governance

Contributions are proposed through pull requests. Alexander Abakah, the lead maintainer,
reviews and merges changes into the official repository and is the only person who
publishes official releases and the official `intentql` package to PyPI.

## License

This documentation project is licensed under the [Apache License 2.0](LICENSE).
