# Open Source

GroundedQL is an open-source semantic compiler for natural-language analytics over Postgres.
The compiler is public so users can inspect, test, extend, and evaluate the full path from
semantic hints to parameterized SQL.

## License

GroundedQL is licensed under the **Apache License 2.0**. The repository's `LICENSE` file is
the authoritative license text.

The BIRD benchmark dataset is maintained by its own authors and uses its own license.
GroundedQL does not bundle downloaded BIRD databases. Follow BIRD's license and attribution
requirements when downloading or redistributing its benchmark material.

## Public Project Scope

The open-source project includes the capabilities that define GroundedQL:

- `QueryIntent` and `QueryPlan` specifications;
- schema introspection and schema configuration;
- evidence and value resolution;
- intent normalization;
- semantic planning and join planning;
- validation and semantic linting;
- deterministic SQL compilation;
- parameterized execution;
- model adapters;
- benchmark runners and generic regression tests.

Users can run GroundedQL locally, connect their own model and Postgres database, and evaluate
the resulting plans and SQL without using an GroundedQL-operated service.

## Governance

GroundedQL is led and maintained by **Alexander Abakah**.

Contributors may propose changes through issues and pull requests. A contribution becomes
part of the official project only after it is reviewed and merged by the lead maintainer.
Only the lead maintainer publishes official releases, creates official release tags, and
publishes the official `groundedql` package to PyPI.

The Apache License 2.0 permits forks and redistribution under its terms. Forks and modified
distributions must not present themselves as official GroundedQL releases.

## Contribution Standard

GroundedQL welcomes contributions that improve general semantic reasoning, compiler
expressiveness, validation, safety, integrations, documentation, and evaluation.

The central technical rule is:

> A benchmark failure may reveal a missing general capability, but benchmark-specific
> behavior does not belong in the compiler.

Good contributions explain the reusable capability being added and include domain-neutral
tests. For example, support for ranked aggregate comparisons is reusable; recognizing a
specific entity from a benchmark database is not.

See [Contributing](contributing.md) for the development workflow and
[Benchmarks](benchmarks.md) for the evaluation methodology.
