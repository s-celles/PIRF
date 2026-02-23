# Portable Integration Rules Format (PIRF)

[![Validate JSON Schemas](https://github.com/s-celles/PIRF/actions/workflows/validate-schemas.yml/badge.svg)](https://github.com/s-celles/PIRF/actions/workflows/validate-schemas.yml)

> **Note**: This repository has been created with AI assistance.

A language-neutral JSON format with validated JSON Schema to represent
all of [RUBI](https://rulebasedintegration.org/)'s ~7,800 symbolic
integration rules and 72,000+ test problems, independently of
Mathematica.

## Overview

PIRF (Portable Integration Rules Format) defines a JSON interchange
format so that any CAS — Julia, Python/SymPy, Java/SymJa, Rust,
JavaScript — can consume RUBI's integration knowledge with a standard
JSON parser. No Mathematica required.

The format covers:

- **Rules**: pattern + constraints + result, with wildcard and predicate
  support
- **Test fixtures**: integrand + variable + optimal antiderivative +
  expected step count
- **Taxonomy**: RUBI's exact 9-section hierarchy
- **Load manifest**: precise file loading order determining rule priority

## Repository Structure

```
PIRF/
├── schemas/                          # JSON Schema definitions (draft-07)
│   ├── pirf-expr.schema.json         # Shared recursive expression type
│   ├── rule-file.schema.json         # Rule file structure
│   ├── test-file.schema.json         # Test fixture structure
│   └── meta.schema.json              # Load manifest structure
├── rules/                            # Integration rules (JSON)
│   ├── meta.json                     # Load manifest, taxonomy, feature flags
│   └── 1-algebraic/                  # Section-based directory tree
│       └── 1.1-binomial/
│           └── 1.1.1-linear/
│               └── 1.1.1.1-(a+b-x)^m.json
├── tests/                            # Test fixtures (JSON)
│   └── 1-algebraic/
│       └── 1.1-binomial/
│           └── 1.1.1-linear/
│               └── 1.1.1.1-(a+b-x)^m.json
├── specifications/
│   └── v0.1.0-draft/
│       └── spec.md                   # EARS specification (159 requirements)
├── rfc.md                            # RFC / design rationale
├── docs/
│   └── schema-validation.md          # Validation documentation
├── validate.sh                       # Local schema validation script
└── .github/workflows/
    └── validate-schemas.yml          # CI schema validation
```

## Quick Start

```bash
pip install check-jsonschema
./validate.sh
```

See [docs/schema-validation.md](docs/schema-validation.md) for details.

## Status

This project is in **early development** (v0.1.0-draft). The EARS
specification, JSON schemas, sample rules, and CI validation pipeline
are in place.

## References

- [RUBI — Rule-based Integration](https://rulebasedintegration.org/)
- [RUBI 5 on GitHub](https://github.com/RuleBasedIntegration/)
- [MathLive MathJSON](https://mathlive.io/math-json/) — operator name
  alignment
- [OpenMath 2.0](https://openmath.org/) — formal semantics mapping

## License

MIT
