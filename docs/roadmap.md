# Project Roadmap

IntentQL's goal is to make natural-language analytics more reliable by moving structural
reasoning from model-generated SQL into a typed semantic compiler.

The roadmap is organized around capabilities, not benchmark-specific cases.

## Now: Broaden Semantic Coverage

- resolve tables, columns, values, and joins from schema evidence;
- improve ranking, extrema, comparisons, ranges, and temporal reasoning;
- support richer aggregate formulas and distinct-count semantics;
- make unsupported requests fail clearly instead of producing plausible wrong SQL;
- protect previous behavior with domain-neutral regression tests.

## Next: Make Evaluation Complete

- run a clean evaluation over all 500 BIRD Mini-Dev cases;
- publish model, configuration, commit SHA, and reproducible commands;
- classify failures by compiler, resolver, model, ambiguity, and infrastructure;
- add more independent datasets and real-world schemas;
- measure accuracy, latency, model cost, and clarification rate separately.

## Next: Improve Developer Experience

- simplify first-run schema generation and review;
- provide clearer plan traces and resolution explanations;
- improve adapter support for local and inexpensive models;
- add examples for embedding IntentQL in APIs and applications;
- document extension interfaces for custom schema metadata and resolvers.

## Later: Production And Enterprise Operations

- native asynchronous execution support;
- richer policy enforcement and audit artifacts;
- evaluation and regression dashboards;
- managed and private deployment options;
- authentication, organizations, RBAC, and enterprise identity;
- connectors beyond Postgres where the compiler's guarantees can be preserved.

## Definition Of Progress

IntentQL is improving when:

1. more useful questions can be represented by the IR;
2. fewer questions require a capable or expensive model;
3. failures become explicit instead of silently wrong;
4. generic improvements transfer across unrelated schemas;
5. users can reproduce the claims we publish.

The implementation-level roadmap also lives in the main repository's
[`ROADMAP.md`](https://github.com/Certifore/intentql/blob/main/ROADMAP.md).
