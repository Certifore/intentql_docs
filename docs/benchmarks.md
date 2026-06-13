# Benchmarks

GroundedQL is evaluated at two different layers:

1. **Compiler properties** test guarantees that should not depend on an LLM.
2. **End-to-end semantic accuracy** tests whether the complete system returns the expected
   result for natural-language questions.

These measurements answer different questions and should not be collapsed into one score.

## Current Results

### Compiler property suite

The repository includes deterministic, no-database checks for core safety properties.

| Property | Current development result | What it measures |
|---|---:|---|
| Injection handling | 50 / 50 | Unknown schema references are rejected and filter values are parameterized |
| Deterministic compilation | 20 / 20 | Recompiling the same plan produces identical SQL |
| Hallucination rejection | 30 / 30 | Unknown tables and columns fail validation |

These are focused tests maintained by the GroundedQL project. They demonstrate architectural
properties, not broad natural-language accuracy.

### BIRD Mini-Dev development evaluation

[BIRD-SQL](https://bird-bench.github.io/) is a cross-domain text-to-SQL benchmark containing
natural-language questions, database schemas, evidence, gold SQL, and executable databases.

The current GroundedQL development run covers a 130-case sequential slice of Mini-Dev
(case IDs 20 through 149):

| Evaluated cases | Exact execution matches | Development accuracy |
|---:|---:|---:|
| 130 | 115 | 88.5% |

!!! warning "This is not an official BIRD leaderboard score"
    This is a partial development evaluation, not the complete 500-case Mini-Dev set.
    It uses BIRD's provided evidence and combines rerun slices after compiler improvements.
    It must not be compared directly with official leaderboard submissions.

## How BIRD Evaluation Works

For each case, the benchmark runner:

1. loads the question, database ID, evidence, and gold SQL;
2. gives the question, schema, and evidence to the configured GroundedQL pipeline;
3. compiles and executes GroundedQL's plan;
4. executes the gold SQL against the same Postgres database;
5. compares the returned values while ignoring row order by default.

GroundedQL is scored on **execution result equality**, not SQL string similarity. Different SQL
is accepted when it returns the same answer.

## How We Treat Mismatches

Every mismatch is counted as a mismatch until it is independently resolved. During
development, mismatches are classified into categories such as:

- missing compiler expressiveness;
- schema or evidence resolution errors;
- ambiguous natural-language questions;
- differing but defensible interpretations;
- potentially inconsistent gold SQL;
- model hint failures;
- execution or infrastructure failures.

Benchmark ambiguity is not used to quietly inflate the score. When a gold query appears
inconsistent with its question, the case remains a reported mismatch unless the benchmark
maintainers or a documented adjudication process establishes otherwise.

## No Reward Hacking

BIRD is a measuring instrument, not the product specification.

Compiler changes made in response to a benchmark failure must express a general semantic
capability. Core logic must not contain:

- BIRD case IDs;
- benchmark database or domain names;
- known question text;
- expected benchmark answers;
- branches that exist only to satisfy one dataset.

Improvements should be supported by domain-neutral regression tests and must preserve
previously correct behavior.

## Reproduce the Property Tests

From the GroundedQL repository:

```bash
python test/test_main.py lint
python test/test_generic_planner_resolution.py
```

## Reproduce BIRD Mini-Dev

BIRD data is downloaded separately and is not bundled with GroundedQL.

```bash
python test/benchmark/bird_minidev.py --dry-run --limit 10
python test/benchmark/bird_minidev.py --limit 25 --llm mistral
```

The benchmark runner, setup instructions, and schema-generation utility live in
`test/benchmark/` in the main repository.

## Publication Standard

Before presenting a complete Mini-Dev result as a project headline, we intend to publish:

- one clean run over all 500 cases;
- exact model and configuration;
- commit SHA;
- execution-comparison policy;
- attempted, correct, wrong, skipped, and infrastructure-failure counts;
- categorized mismatches;
- reproducible commands.

Until then, partial scores are labeled as development results.
