# Changelog

All notable changes to this project will be documented in this file.

The format is based on [Keep a Changelog](https://keepachangelog.com/en/1.1.0/),
and this project adheres to [Semantic Versioning](https://semver.org/spec/v2.0.0.html).

## [Unreleased]

### Added

- EARS specification (159 requirements) for PIRF v0.1.0-draft
- RFC describing the design rationale and format overview
- 4 JSON Schema files (draft-07): `pirf-expr`, `rule-file`, `test-file`, `meta`
- Sample rule file with 3 linear binomial rules (RUBI Section 1.1.1)
- Sample test fixture file with 3 corresponding test problems
- Load manifest (`rules/meta.json`) with 9-section RUBI taxonomy
- GitHub Actions workflow for CI schema validation
- Local validation script (`validate.sh`)
- Schema validation documentation (`docs/schema-validation.md`)
