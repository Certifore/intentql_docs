# Contributing

Thanks for your interest in contributing to GroundedQL!

!!! tip "New to the codebase?"
    Read [Why a Compiler?](architecture.md) and [Core Concepts](concepts.md) first to
    understand the trust boundary and current pipeline.

## The Generality Rule

GroundedQL uses benchmarks to discover missing capabilities, not as a set of answers to
memorize.

Every compiler or resolver change must describe a reusable semantic capability and include
domain-neutral regression coverage. Core logic must not branch on:

- benchmark case IDs;
- benchmark database or domain names;
- customer or entity names;
- known question text;
- expected answers.

A useful test is: **would this change still make sense if every table, column, and entity
were renamed?** If not, the behavior probably belongs in schema configuration, runtime
evidence, or an application extension rather than the core compiler.

When fixing a benchmark mismatch, include:

1. the general failure category;
2. a domain-neutral regression test;
3. evidence that previously correct behavior remains correct;
4. any ambiguity or assumptions the compiler cannot safely resolve.

---

## Development Setup

### 1. Clone the Repository

```bash
git clone https://github.com/Certifore/groundedql
cd groundedql
```

### 2. Create a Virtual Environment

```bash
python3 -m venv env_dsl
source env_dsl/bin/activate   # Linux/macOS
# or: env_dsl\Scripts\activate  # Windows
```

### 3. Install in Editable Mode

```bash
pip install -e ".[dev]"
```

This installs `groundedql` in editable mode (source changes are reflected immediately), plus `ruff`, `build`, and `twine`.

### 4. Install LLM Extras (optional)

```bash
pip install -e ".[dev,openai]"          # OpenAI SDK
pip install langchain-google-genai      # Gemini via LangChain
pip install langchain-groq              # Groq via LangChain
pip install langchain-ollama            # Ollama (local models)
```

---

## Project Structure

```
groundedql/
  __init__.py           # public API — everything exported from here
  compiler.py           # core SQL compiler: logical → physical, values → bind params
  executor.py           # executes compiled SQL against Postgres
  planner.py            # LLM → QueryPlan orchestration + retry loop
  intent_planner.py     # semantic-hint extraction and deterministic plan orchestration
  evidence_planner.py   # generic evidence-guided deterministic planning
  intent_normalize.py   # deterministic intent normalization
  validation.py         # three-layer validation: Pydantic + allowlist + semantic
  join_planner.py       # BFS auto-join injection
  semantic_lint.py      # question-aware lint checks (5 rules)
  spec_builder.py       # auto-generates LLM prompt spec from schema.yaml
  queryplan_models.py   # Pydantic models for the QueryPlan DSL
  llm_client.py         # LLMClient protocol + CallableLLMClient
  llm_adapters.py       # OpenAI and LangChain adapters + make_llm_client() factory
  exceptions.py         # structured exception hierarchy
  agent.py              # high-level QueryAgent (planner + executor combined)
  api/
    api.py              # execute_query_plan, validate_query_plan, load_and_validate_schema
    spec_api.py         # get_queryplan_instructions

config/
  schema.yaml                     # demo Northwind schema
  queryplan_spec.yaml             # hand-authored reference spec
  queryplan_spec_generated.yaml   # auto-generated — regenerate after schema changes

test/
  test_main.py                         # unit and integration tests
  test_generic_planner_resolution.py   # domain-neutral planner regression tests
  regression_test/                     # regression fixtures
  benchmark/                           # BIRD Mini-Dev runner and utilities
```

---

## Running Tests

All automated checks go through `test/test_main.py` (no separate pytest modules). Cases live in `test/regression_test/test_qs.json` with a `type` field: **`db`** (default), **`lint`**, **`canonical`**, or **`compile`** (compiler / schema / validation; no DB).

**No-DB tests** — semantic lint, canonicalize/fingerprint, and compile rows:

```bash
python test/test_main.py lint
```

**Regression suite** — executes `db` rows from `test/regression_test/test_qs.json` against Postgres (plus non-DB rows in the same file):

```bash
cp test/.env.example test/.env
# fill in: DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD
```

```bash
python test/test_main.py          # full suite (default)
python test/test_main.py run      # same: execute and print results
python test/test_main.py check    # compare to saved baseline (CI)
```

`pipeline` mode runs compile tests from the same JSON as a preflight, then the pipeline benchmark.

---

## Linting and Formatting

GroundedQL uses [Ruff](https://docs.astral.sh/ruff/):

```bash
ruff check groundedql/         # check for issues
ruff check --fix groundedql/   # auto-fix
ruff format groundedql/        # format
```

All configuration lives in `pyproject.toml`.

---

## Regenerating the Spec

After any change to `schema.yaml`:

```bash
python -m groundedql.spec_builder \
    --schema config/schema.yaml \
    --output config/queryplan_spec_generated.yaml
```

---

## Common Extension Patterns

=== "New Operator"

    1. Add the literal to `Operator` in `queryplan_models.py`
    2. Handle it in `Compiler._compile_bool_expr` in `compiler.py`
    3. Add it to `_operators_block()` in `spec_builder.py`
    4. Add a `db` (or `compile`, etc.) case in `test/regression_test/test_qs.json`

=== "New Aggregation"

    1. Add the literal to `Agg` in `queryplan_models.py`
    2. Handle it in `Compiler._build_select_query` in `compiler.py`
    3. Add a case in `test/regression_test/test_qs.json`

=== "New Semantic Lint Rule"

    1. Add a `_check_*` function in `semantic_lint.py`
    2. Call it from `semantic_lint()`
    3. Add a test that provides a question with the pattern and a plan that violates the rule

---

## Submitting a Pull Request

1. **Fork** and create a feature branch:

    ```bash
    git checkout -b feat/my-feature
    ```

2. **Write tests** for your changes — all PRs require coverage for new behaviour.

3. **Run checks:**

    ```bash
    ruff check groundedql/
    python test/test_main.py lint
    python test/test_generic_planner_resolution.py
    ```

4. **Commit** with a clear message:

    ```bash
    git commit -m "feat: add support for BETWEEN operator in compiler"
    ```

5. **Open a PR** against `main`. Describe what the change does and why.

Pull requests are proposals. The lead maintainer, Alexander Abakah, reviews and decides
whether to merge every change into the official repository. Opening a pull request or
contributing code does not grant merge or release authority.

---

## Building the Package

```bash
python -m build
# output: dist/
```

Official GitHub releases, release tags, and PyPI publications are created only by the lead
maintainer.

---

## Docs Locally

```bash
pip install -r requirements-docs.txt
mkdocs serve --dev-addr 127.0.0.1:8001
```

Open [http://127.0.0.1:8001](http://127.0.0.1:8001) to preview.

```bash
mkdocs build   # static HTML → site/
```

---

## Code Style Guidelines

- Type annotations on all public functions and constructors
- Keep functions focused — the compiler's `_compile_*` methods are a good model
- Raise typed exceptions (`QueryPlanError`, `SchemaError`, …) — never bare `Exception` from public paths
- All filter values must go through bind parameters — never `str.format()` or f-strings into SQL
- Tests should be self-contained and not require a live DB unless they're in `regression_test/`
- Benchmark-driven changes must remain domain-neutral and must not encode expected answers

## License

By submitting a contribution for inclusion, you agree that it may be distributed under
the project's Apache License 2.0.
