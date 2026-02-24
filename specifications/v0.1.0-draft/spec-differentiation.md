# EARS Specification — Portable Format for Symbolic Differentiation Rules

**Project:** RUBI-PIRF Differentiation Extension (PIRF-D)
**Version:** 0.1.0-draft
**Date:** 2026-02-24
**Method:** EARS (Easy Approach to Requirements Syntax) — Alistair Mavin, 2009
**System:** Any CAS implementing rule-based symbolic differentiation
**Companion to:** PIRF v0.1.0-draft (Portable Integration Rules Format)

---

## 1. Purpose and scope

This document specifies the requirements for extending the **PIRF format**
to represent **symbolic differentiation rules** in the same portable,
language-neutral JSON format defined by the PIRF integration specification.

Where PIRF captures the ~7,800 rules of RUBI for integration, this
extension captures the **differentiation rules** required by a symbolic
computation engine. Differentiation rules are substantially simpler and
fewer in number than integration rules, but they are:

1. **Essential to RUBI itself** — many integration rules invoke the
   derivative operator `D` or RUBI's own `Dif` in constraints and results.
2. **A natural complement** — any CAS consuming PIRF integration rules
   also needs differentiation for constraint evaluation and simplification.
3. **Currently implicit** — differentiation is assumed to be a CAS built-in,
   but a portable rule set should make these rules explicit and portable.

### 1.1 Relationship to the PIRF integration specification

This specification is a **strict extension** of PIRF v0.1.0-draft. It:

- **Reuses** PIRF-Expr (§4 of the integration spec) without modification.
- **Reuses** the JSON file format (§5), wildcards (§6), constraints (§7),
  and loader (§9) exactly as defined.
- **Adds** a new taxonomy section (Section 10: Differentiation rules) and
  differentiation-specific operators.
- **Does not modify** any existing PIRF requirement.

### 1.2 Scope of differentiation rules

This specification covers:

- **Elementary differentiation rules**: power rule, sum rule, product rule,
  quotient rule, chain rule.
- **Transcendental function derivatives**: trigonometric, inverse trigonometric,
  hyperbolic, inverse hyperbolic, exponential, logarithmic.
- **Special function derivatives**: derivatives of the special functions
  catalogued in PIRF-X-016 (Gamma, Beta, Bessel, error functions, etc.).
- **Structural rules**: linearity, constant factor, derivative of a constant.
- **Higher-order derivatives**: iterated differentiation D^n.
- **Partial derivatives**: derivative with respect to one variable in
  multivariate expressions.
- **Implicit differentiation**: not covered (engine-level, not rule-level).

### 1.3 Rule count estimate

Standard differentiation involves approximately **80–120 rules** for full
coverage of the PIRF operator catalogue, compared to ~7,800 integration rules.
This reflects the algorithmic nature of differentiation (every elementary
function has an explicit derivative formula) versus the heuristic nature
of integration.

---

## 2. Glossary (additions to PIRF §2)

| Term | Definition |
|------|-----------|
| **PIRF-D** | PIRF Differentiation extension — the extension specified herein |
| **Differentiation rule** | Triple (pattern, constraints, result) describing a derivative transformation |
| **Chain rule** | Meta-rule: D[f(g(x)), x] = f'(g(x)) · g'(x), applied recursively |
| **Leibniz notation** | D[f, x] denotes df/dx |
| **Higher-order derivative** | D[D[f, x], x] or equivalently D[f, {x, n}] |
| **Dif** | RUBI's own derivative operator, distinct from the CAS built-in `D` to avoid side effects |

---

## 3. EARS patterns used

| Pattern | Keyword | Usage |
|---------|---------|-------|
| Ubiquitous | *(none)* | Properties that are always true |
| Event-driven | **When** | Response to a triggering event |
| State-driven | **While** | Active as long as a state persists |
| Optional feature | **Where** | Applies if a feature is present |
| Unwanted behaviour | **If…then** | Error handling or abnormal situations |
| Complex | **While…When** | State + event combination |

---

## 4. Requirements — Differentiation-specific operators

### 4.1 Core differentiation operators

**PIRF-D-X-001** [Ubiquitous]
PIRF-Expr shall define the following **differentiation operators**:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `D` | 2 | Derivative of $1 with respect to $2: d($1)/d($2) |
| `D` | 3 | n-th derivative of $1 with respect to $2, with order $3: d^n($1)/d($2)^n |
| `Dif` | 2 | RUBI's own derivative: Dif($1, $2) — same semantics as D but avoids built-in CAS side effects |
| `Dt` | 1-2 | Total derivative: Dt($1) or Dt($1, $2) for total derivative with respect to $2 |

> **Design note — D vs Dif.** RUBI defines `Dif` as a separate derivative
> operator to prevent the host CAS from applying its own differentiation
> rules (which may produce forms unsuitable for pattern matching). The
> PIRF differentiation rules should be applied via `Dif`; the operator `D`
> is used when the CAS built-in behaviour is acceptable. A Phrasebook may
> map both to the same implementation if the CAS has no auto-evaluation.

**PIRF-D-X-002** [Ubiquitous]
The `D` operator with arity 2 shall denote a first-order derivative:
`["D", f, x]` represents df/dx.

**PIRF-D-X-003** [Ubiquitous]
The `D` operator with arity 3 shall denote a higher-order derivative:
`["D", f, x, n]` represents d^n f / dx^n, where n is a positive integer.

> **Design note — Higher-order derivative encoding.**
> An alternative encoding would be `["D", f, ["List", x, n]]`
> (Mathematica style). This specification uses the flat form `["D", f, x, n]`
> for simplicity. The Phrasebook for Mathematica shall translate between
> the two forms.

### 4.2 Relationship to existing PIRF operators

**PIRF-D-X-010** [Ubiquitous]
The operators `D` and `Dif` defined in PIRF §4.11.6 (Calculus and general)
shall have the semantics defined in this section. This differentiation
extension provides the **rules** that implement these operators; the
integration spec defines only their existence and calling convention.

**PIRF-D-X-011** [Ubiquitous]
Differentiation rules shall use the same PIRF-Expr expression language,
wildcard syntax, and constraint predicates as integration rules, without
any extension to the expression grammar.

---

## 5. Requirements — Differentiation rule file format

### 5.1 File structure

**PIRF-D-F-001** [Ubiquitous]
Differentiation rule files shall follow the same JSON structure as
integration rule files (PIRF §5.1): a root JSON object with mandatory
fields `$schema`, `section`, `title`, and `rules`.

**PIRF-D-F-002** [Ubiquitous]
Each differentiation rule object shall contain the mandatory fields
`id` (unique integer), `pattern` (PIRF-Expr), `constraints` (array of
PIRF-Expr), and `result` (PIRF-Expr).

**PIRF-D-F-003** [Ubiquitous]
Differentiation rule IDs shall be unique within the differentiation
rule set and shall not collide with integration rule IDs. The ID
range 100001–199999 is reserved for differentiation rules.

> **Design note — ID range.** RUBI integration rules use IDs in the range
> 1–~7800. Reserving 100001+ for differentiation avoids collision and
> allows a merged rule set with both integration and differentiation rules.

### 5.2 File organisation

**PIRF-D-F-010** [Ubiquitous]
Differentiation rule files shall be placed in a `deriv/` directory tree
at the same level as the `rules/` directory for integration:

```
PIRF/
├── rules/              # Integration rules (existing)
│   ├── meta.json
│   └── 1-algebraic/
│       └── ...
├── deriv/              # Differentiation rules (new)
│   ├── meta.json       # Differentiation load manifest
│   ├── 10.1-structural/
│   │   └── 10.1.1-linearity.json
│   ├── 10.2-elementary/
│   │   ├── 10.2.1-power.json
│   │   ├── 10.2.2-exponential.json
│   │   └── 10.2.3-logarithmic.json
│   ├── 10.3-trigonometric/
│   │   ├── 10.3.1-circular.json
│   │   └── 10.3.2-inverse-circular.json
│   ├── 10.4-hyperbolic/
│   │   ├── 10.4.1-hyperbolic.json
│   │   └── 10.4.2-inverse-hyperbolic.json
│   ├── 10.5-special-functions/
│   │   ├── 10.5.1-gamma-beta.json
│   │   ├── 10.5.2-error-functions.json
│   │   ├── 10.5.3-elliptic.json
│   │   ├── 10.5.4-bessel.json
│   │   ├── 10.5.5-integral-functions.json
│   │   └── 10.5.6-hypergeometric.json
│   ├── 10.6-combinatorial/
│   │   └── 10.6.1-product-quotient-chain.json
│   └── 10.7-higher-order/
│       └── 10.7.1-iterated.json
└── tests/              # Existing test fixtures
```

**PIRF-D-F-011** [Ubiquitous]
The differentiation `meta.json` shall contain a `"load_order"` array
specifying the exact loading sequence, following the same schema as the
integration `meta.json`.

**PIRF-D-F-012** [Ubiquitous]
Structural rules (linearity, constant) shall be loaded before elementary
function rules, and elementary function rules before composite rules
(product, quotient, chain), to establish correct rule priority.

---

## 6. Requirements — Differentiation rule taxonomy

**PIRF-D-T-001** [Ubiquitous]
The differentiation rules shall be organised in the following taxonomy
as Section 10 of the unified PIRF hierarchy:

| Section | Title | Description |
|---------|-------|-------------|
| 10.1 | Structural rules | Constant, linearity, sum |
| 10.2 | Elementary function derivatives | Power, exponential, logarithmic |
| 10.3 | Trigonometric derivatives | Circular and inverse circular |
| 10.4 | Hyperbolic derivatives | Hyperbolic and inverse hyperbolic |
| 10.5 | Special function derivatives | Gamma, Beta, Bessel, Erf, elliptic, etc. |
| 10.6 | Combinatorial rules | Product rule, quotient rule, chain rule |
| 10.7 | Higher-order derivatives | Iterated and n-th derivatives |

**PIRF-D-T-002** [Ubiquitous]
Section numbering shall start at 10 to avoid collision with the integration
taxonomy (sections 1–9).

---

## 7. Requirements — Structural differentiation rules (§10.1)

### 7.1 Constant and identity

**PIRF-D-R-001** [Ubiquitous]
The system shall implement the **constant rule**: when `c` is free of `x`,
then `D[c, x] = 0`.

```json
{
  "id": 100001,
  "pattern": ["D", "c_", "x_"],
  "constraints": [["FreeQ", "c", "x"]],
  "result": 0
}
```

**PIRF-D-R-002** [Ubiquitous]
The system shall implement the **identity rule**: `D[x, x] = 1`.

```json
{
  "id": 100002,
  "pattern": ["D", "x_", "x_"],
  "constraints": [],
  "result": 1
}
```

### 7.2 Linearity

**PIRF-D-R-003** [Ubiquitous]
The system shall implement the **sum rule**:
`D[Add[f, g, ...], x] = Add[D[f, x], D[g, x], ...]`.

```json
{
  "id": 100003,
  "pattern": ["D", ["Add", "f_", "g_"], "x_"],
  "constraints": [],
  "result": ["Add", ["D", "f", "x"], ["D", "g", "x"]]
}
```

**PIRF-D-R-004** [Ubiquitous]
The system shall implement the **constant multiple rule**: when `c` is
free of `x`, then `D[Multiply[c, f], x] = Multiply[c, D[f, x]]`.

```json
{
  "id": 100004,
  "pattern": ["D", ["Multiply", "c_", "f_"], "x_"],
  "constraints": [["FreeQ", "c", "x"]],
  "result": ["Multiply", "c", ["D", "f", "x"]]
}
```

**PIRF-D-R-005** [Ubiquitous]
The system shall implement the **subtraction rule**:
`D[Subtract[f, g], x] = Subtract[D[f, x], D[g, x]]`.

**PIRF-D-R-006** [Ubiquitous]
The system shall implement the **negation rule**:
`D[Negate[f], x] = Negate[D[f, x]]`.

---

## 8. Requirements — Elementary function derivative rules (§10.2)

### 8.1 Power rule

**PIRF-D-R-010** [Ubiquitous]
The system shall implement the **power rule** with constant exponent:
when `n` is free of `x`, then
`D[Power[x, n], x] = Multiply[n, Power[x, Subtract[n, 1]]]`.

```json
{
  "id": 100010,
  "pattern": ["D", ["Power", "x_", "n_"], "x_"],
  "constraints": [["FreeQ", "n", "x"]],
  "result": ["Multiply", "n", ["Power", "x", ["Subtract", "n", 1]]]
}
```

**PIRF-D-R-011** [Ubiquitous]
The system shall implement the **general power rule** with chain rule:
when `n` is free of `x`, then
`D[Power[f, n], x] = Multiply[n, Power[f, Subtract[n, 1]], D[f, x]]`.

```json
{
  "id": 100011,
  "pattern": ["D", ["Power", "f_", "n_"], "x_"],
  "constraints": [["FreeQ", "n", "x"], ["Not", ["FreeQ", "f", "x"]]],
  "result": ["Multiply", "n", ["Power", "f", ["Subtract", "n", 1]], ["D", "f", "x"]]
}
```

**PIRF-D-R-012** [Ubiquitous]
The system shall implement the **exponential power rule** (variable in base
and exponent): `D[Power[f, g], x] = Multiply[Power[f, g], Add[Multiply[D[g, x], Log[f]], Multiply[g, Divide[D[f, x], f]]]]`.

```json
{
  "id": 100012,
  "pattern": ["D", ["Power", "f_", "g_"], "x_"],
  "constraints": [
    ["Not", ["FreeQ", "f", "x"]],
    ["Not", ["FreeQ", "g", "x"]]
  ],
  "result": ["Multiply",
    ["Power", "f", "g"],
    ["Add",
      ["Multiply", ["D", "g", "x"], ["Log", "f"]],
      ["Multiply", "g", ["Divide", ["D", "f", "x"], "f"]]
    ]
  ]
}
```

**PIRF-D-R-013** [Ubiquitous]
The system shall implement the **sqrt derivative**:
`D[Sqrt[f], x] = Divide[D[f, x], Multiply[2, Sqrt[f]]]`.

### 8.2 Exponential rules

**PIRF-D-R-020** [Ubiquitous]
The system shall implement the **natural exponential rule**:
`D[Exp[f], x] = Multiply[Exp[f], D[f, x]]`.

```json
{
  "id": 100020,
  "pattern": ["D", ["Exp", "f_"], "x_"],
  "constraints": [],
  "result": ["Multiply", ["Exp", "f"], ["D", "f", "x"]]
}
```

**PIRF-D-R-021** [Ubiquitous]
The system shall implement the **general exponential rule**: when `a` is
free of `x`, then
`D[Power[a, f], x] = Multiply[Power[a, f], Log[a], D[f, x]]`.

```json
{
  "id": 100021,
  "pattern": ["D", ["Power", "a_", "f_"], "x_"],
  "constraints": [["FreeQ", "a", "x"], ["Not", ["FreeQ", "f", "x"]]],
  "result": ["Multiply", ["Power", "a", "f"], ["Log", "a"], ["D", "f", "x"]]
}
```

### 8.3 Logarithmic rules

**PIRF-D-R-030** [Ubiquitous]
The system shall implement the **natural logarithm rule**:
`D[Log[f], x] = Divide[D[f, x], f]`.

```json
{
  "id": 100030,
  "pattern": ["D", ["Log", "f_"], "x_"],
  "constraints": [],
  "result": ["Divide", ["D", "f", "x"], "f"]
}
```

**PIRF-D-R-031** [Ubiquitous]
The system shall implement the **general logarithm rule** (Log base b):
when `b` is free of `x`, then
`D[Log[b, f], x] = Divide[D[f, x], Multiply[f, Log[b]]]`.

```json
{
  "id": 100031,
  "pattern": ["D", ["Log", "b_", "f_"], "x_"],
  "constraints": [["FreeQ", "b", "x"]],
  "result": ["Divide", ["D", "f", "x"], ["Multiply", "f", ["Log", "b"]]]
}
```

---

## 9. Requirements — Trigonometric derivative rules (§10.3)

### 9.1 Circular trigonometric functions

**PIRF-D-R-040** [Ubiquitous]
The system shall implement the following **circular trig derivative rules**,
each with automatic chain rule application via `D[f, x]`:

| ID | Rule | Formula |
|----|------|---------|
| 100040 | D[Sin[f], x] | Multiply[Cos[f], D[f, x]] |
| 100041 | D[Cos[f], x] | Multiply[Negate[Sin[f]], D[f, x]] |
| 100042 | D[Tan[f], x] | Multiply[Power[Sec[f], 2], D[f, x]] |
| 100043 | D[Cot[f], x] | Multiply[Negate[Power[Csc[f], 2]], D[f, x]] |
| 100044 | D[Sec[f], x] | Multiply[Sec[f], Tan[f], D[f, x]] |
| 100045 | D[Csc[f], x] | Multiply[Negate[Csc[f]], Cot[f], D[f, x]] |

**Example (Sin):**

```json
{
  "id": 100040,
  "pattern": ["D", ["Sin", "f_"], "x_"],
  "constraints": [],
  "result": ["Multiply", ["Cos", "f"], ["D", "f", "x"]]
}
```

### 9.2 Inverse circular trigonometric functions

**PIRF-D-R-050** [Ubiquitous]
The system shall implement the following **inverse trig derivative rules**:

| ID | Rule | Formula |
|----|------|---------|
| 100050 | D[Asin[f], x] | Divide[D[f, x], Sqrt[Subtract[1, Power[f, 2]]]] |
| 100051 | D[Acos[f], x] | Negate[Divide[D[f, x], Sqrt[Subtract[1, Power[f, 2]]]]] |
| 100052 | D[Atan[f], x] | Divide[D[f, x], Add[1, Power[f, 2]]] |
| 100053 | D[Acot[f], x] | Negate[Divide[D[f, x], Add[1, Power[f, 2]]]] |
| 100054 | D[Asec[f], x] | Divide[D[f, x], Multiply[Abs[f], Sqrt[Subtract[Power[f, 2], 1]]]] |
| 100055 | D[Acsc[f], x] | Negate[Divide[D[f, x], Multiply[Abs[f], Sqrt[Subtract[Power[f, 2], 1]]]]] |

**Example (Atan):**

```json
{
  "id": 100052,
  "pattern": ["D", ["Atan", "f_"], "x_"],
  "constraints": [],
  "result": ["Divide", ["D", "f", "x"], ["Add", 1, ["Power", "f", 2]]]
}
```

---

## 10. Requirements — Hyperbolic derivative rules (§10.4)

### 10.1 Hyperbolic functions

**PIRF-D-R-060** [Ubiquitous]
The system shall implement the following **hyperbolic derivative rules**:

| ID | Rule | Formula |
|----|------|---------|
| 100060 | D[Sinh[f], x] | Multiply[Cosh[f], D[f, x]] |
| 100061 | D[Cosh[f], x] | Multiply[Sinh[f], D[f, x]] |
| 100062 | D[Tanh[f], x] | Multiply[Power[Sech[f], 2], D[f, x]] |
| 100063 | D[Coth[f], x] | Multiply[Negate[Power[Csch[f], 2]], D[f, x]] |
| 100064 | D[Sech[f], x] | Multiply[Negate[Sech[f]], Tanh[f], D[f, x]] |
| 100065 | D[Csch[f], x] | Multiply[Negate[Csch[f]], Coth[f], D[f, x]] |

### 10.2 Inverse hyperbolic functions

**PIRF-D-R-070** [Ubiquitous]
The system shall implement the following **inverse hyperbolic derivative rules**:

| ID | Rule | Formula |
|----|------|---------|
| 100070 | D[Asinh[f], x] | Divide[D[f, x], Sqrt[Add[Power[f, 2], 1]]] |
| 100071 | D[Acosh[f], x] | Divide[D[f, x], Sqrt[Subtract[Power[f, 2], 1]]] |
| 100072 | D[Atanh[f], x] | Divide[D[f, x], Subtract[1, Power[f, 2]]] |
| 100073 | D[Acoth[f], x] | Divide[D[f, x], Subtract[1, Power[f, 2]]] |
| 100074 | D[Asech[f], x] | Negate[Divide[D[f, x], Multiply[f, Sqrt[Subtract[1, Power[f, 2]]]]]] |
| 100075 | D[Acsch[f], x] | Negate[Divide[D[f, x], Multiply[Abs[f], Sqrt[Add[1, Power[f, 2]]]]]] |

---

## 11. Requirements — Special function derivative rules (§10.5)

### 11.1 Error functions

**PIRF-D-R-080** [Ubiquitous]
The system shall implement the following **error function derivative rules**:

| ID | Rule | Formula |
|----|------|---------|
| 100080 | D[Erf[f], x] | Multiply[Divide[2, Sqrt[Pi]], Exp[Negate[Power[f, 2]]], D[f, x]] |
| 100081 | D[Erfc[f], x] | Negate of D[Erf[f], x] |
| 100082 | D[Erfi[f], x] | Multiply[Divide[2, Sqrt[Pi]], Exp[Power[f, 2]], D[f, x]] |

### 11.2 Gamma and Beta functions

**PIRF-D-R-085** [Ubiquitous]
The system shall implement the following **Gamma/Beta derivative rules**:

| ID | Rule | Formula |
|----|------|---------|
| 100085 | D[Gamma[f], x] | Multiply[Gamma[f], PolyGamma[0, f], D[f, x]] |
| 100086 | D[LogGamma[f], x] | Multiply[PolyGamma[0, f], D[f, x]] |

> **Design note — PolyGamma.** The derivative of Gamma requires the
> PolyGamma (digamma) function ψ(z) = Γ'(z)/Γ(z). This operator is
> added to the PIRF-Expr catalogue by this extension:
>
> | Operator | Arity | Semantics |
> |----------|-------|-----------|
> | `PolyGamma` | 2 | Polygamma function ψ^(n)(z) |

**PIRF-D-R-087** [Ubiquitous]
The system shall implement the **incomplete Gamma derivative**:
`D[Gamma[a, f], x] = Multiply[Negate[Power[f, Subtract[a, 1]]], Exp[Negate[f]], D[f, x]]`,
where `a` is free of `x`.

### 11.3 Bessel functions

**PIRF-D-R-090** [Ubiquitous]
The system shall implement the following **Bessel function derivative rules**,
where `n` is free of `x`:

| ID | Rule | Formula |
|----|------|---------|
| 100090 | D[BesselJ[n, f], x] | Multiply[Divide[1, 2], Subtract[BesselJ[Subtract[n, 1], f], BesselJ[Add[n, 1], f]], D[f, x]] |
| 100091 | D[BesselY[n, f], x] | Multiply[Divide[1, 2], Subtract[BesselY[Subtract[n, 1], f], BesselY[Add[n, 1], f]], D[f, x]] |
| 100092 | D[BesselI[n, f], x] | Multiply[Divide[1, 2], Add[BesselI[Subtract[n, 1], f], BesselI[Add[n, 1], f]], D[f, x]] |
| 100093 | D[BesselK[n, f], x] | Multiply[Negate[Divide[1, 2]], Add[BesselK[Subtract[n, 1], f], BesselK[Add[n, 1], f]], D[f, x]] |

### 11.4 Elliptic integrals

**PIRF-D-R-095** [Ubiquitous]
The system shall implement derivative rules for the elliptic integral
functions (`EllipticF`, `EllipticE`, `EllipticK`, `EllipticPi`) as defined
in PIRF-X-016, with respect to their arguments.

| ID | Rule | Formula |
|----|------|---------|
| 100095 | D[EllipticK[f], x] | Multiply[Divide[Subtract[EllipticE[f], Multiply[Subtract[1, f], EllipticK[f]]], Multiply[2, f, Subtract[1, f]]], D[f, x]] |
| 100096 | D[EllipticE[f], x] | Multiply[Divide[Subtract[EllipticE[f], EllipticK[f]], Multiply[2, f]], D[f, x]] |

### 11.5 Polylogarithm

**PIRF-D-R-098** [Ubiquitous]
The system shall implement the **polylogarithm derivative**:
when `n` is free of `x`, then
`D[PolyLog[n, f], x] = Divide[Multiply[PolyLog[Subtract[n, 1], f], D[f, x]], f]`.

### 11.6 Integral functions

**PIRF-D-R-100** [Ubiquitous]
The system shall implement derivative rules for the integral functions
defined in PIRF-X-016:

| ID | Rule | Formula |
|----|------|---------|
| 100100 | D[ExpIntegralEi[f], x] | Multiply[Divide[Exp[f], f], D[f, x]] |
| 100101 | D[LogIntegral[f], x] | Multiply[Divide[1, Log[f]], D[f, x]] |
| 100102 | D[SinIntegral[f], x] | Multiply[Divide[Sin[f], f], D[f, x]] |
| 100103 | D[CosIntegral[f], x] | Multiply[Divide[Cos[f], f], D[f, x]] |
| 100104 | D[SinhIntegral[f], x] | Multiply[Divide[Sinh[f], f], D[f, x]] |
| 100105 | D[CoshIntegral[f], x] | Multiply[Divide[Cosh[f], f], D[f, x]] |
| 100106 | D[FresnelS[f], x] | Multiply[Sin[Multiply[Divide["Pi", 2], Power[f, 2]]], D[f, x]] |
| 100107 | D[FresnelC[f], x] | Multiply[Cos[Multiply[Divide["Pi", 2], Power[f, 2]]], D[f, x]] |

### 11.7 Hypergeometric functions

**PIRF-D-R-110** [Ubiquitous]
The system shall implement the **Gauss hypergeometric derivative** with
respect to the last argument: when `a`, `b`, `c` are free of `x`, then
`D[Hypergeometric2F1[a, b, c, f], x] = Multiply[Divide[Multiply[a, b], c], Hypergeometric2F1[Add[a, 1], Add[b, 1], Add[c, 1], f], D[f, x]]`.

### 11.8 Product logarithm (Lambert W)

**PIRF-D-R-115** [Ubiquitous]
The system shall implement the **Lambert W derivative**:
`D[ProductLog[f], x] = Divide[Multiply[ProductLog[f], D[f, x]], Multiply[f, Add[1, ProductLog[f]]]]`.

---

## 12. Requirements — Combinatorial rules (§10.6)

### 12.1 Product rule

**PIRF-D-R-120** [Ubiquitous]
The system shall implement the **product rule**:
`D[Multiply[f, g], x] = Add[Multiply[D[f, x], g], Multiply[f, D[g, x]]]`.

```json
{
  "id": 100120,
  "pattern": ["D", ["Multiply", "f_", "g_"], "x_"],
  "constraints": [
    ["Not", ["FreeQ", "f", "x"]],
    ["Not", ["FreeQ", "g", "x"]]
  ],
  "result": ["Add",
    ["Multiply", ["D", "f", "x"], "g"],
    ["Multiply", "f", ["D", "g", "x"]]
  ]
}
```

> **Design note — Product rule priority.** The constant multiple rule
> (PIRF-D-R-004) must be loaded before the product rule so that
> `D[Multiply[c, f], x]` with constant `c` is simplified to
> `Multiply[c, D[f, x]]` rather than expanded via the full product rule.

### 12.2 Quotient rule

**PIRF-D-R-121** [Ubiquitous]
The system shall implement the **quotient rule**:
`D[Divide[f, g], x] = Divide[Subtract[Multiply[D[f, x], g], Multiply[f, D[g, x]]], Power[g, 2]]`.

```json
{
  "id": 100121,
  "pattern": ["D", ["Divide", "f_", "g_"], "x_"],
  "constraints": [
    ["Not", ["FreeQ", "g", "x"]]
  ],
  "result": ["Divide",
    ["Subtract",
      ["Multiply", ["D", "f", "x"], "g"],
      ["Multiply", "f", ["D", "g", "x"]]
    ],
    ["Power", "g", 2]
  ]
}
```

### 12.3 Chain rule

**PIRF-D-R-122** [Ubiquitous]
The chain rule shall be implemented **implicitly** by including `D[f, x]`
in the result of every function derivative rule (as shown in §8–11).
When the Engine encounters `D[f, x]` in a result, it shall recursively
apply differentiation rules to `f`.

**PIRF-D-R-123** [Ubiquitous]
The Engine shall apply the chain rule recursively until all `D` operators
are resolved (i.e., the result contains no residual `D` operators), or
until the expression is free of the differentiation variable.

---

## 13. Requirements — Higher-order derivatives (§10.7)

**PIRF-D-R-130** [Event-driven]
When the Engine encounters `["D", f, x, n]` with integer `n > 1`, the
Engine shall reduce it to `["D", ["D", f, x, ["Subtract", n, 1]], x]`
(i.e., apply the derivative recursively n times).

**PIRF-D-R-131** [Event-driven]
When the Engine encounters `["D", f, x, 1]`, the Engine shall treat it
as `["D", f, x]`.

**PIRF-D-R-132** [Event-driven]
When the Engine encounters `["D", f, x, 0]`, the Engine shall return `f`
unchanged.

**PIRF-D-R-133** [Unwanted behaviour]
If `n` is not a non-negative integer, then the Engine shall return the
expression in unevaluated form and emit a warning.

---

## 14. Requirements — Differentiation engine

### 14.1 Evaluation strategy

**PIRF-D-E-001** [Event-driven]
When the Engine receives an expression `["D", f, x]` to evaluate, the
Engine shall traverse differentiation rules in load-manifest priority order
and apply the first matching rule.

**PIRF-D-E-002** [Event-driven]
When the result of a differentiation rule contains residual `["D", g, x]`
sub-expressions, the Engine shall recursively evaluate each such
sub-expression.

**PIRF-D-E-003** [Ubiquitous]
The Engine shall apply differentiation rules **bottom-up** (innermost
derivatives first) to ensure correct chain rule application.

### 14.2 Recursion safety

**PIRF-D-E-010** [Ubiquitous]
The Engine shall enforce a configurable maximum recursion depth
(default: 50) for differentiation.

**PIRF-D-E-011** [Unwanted behaviour]
If recursion exceeds the limit, then the Engine shall halt and return the
derivative in unevaluated form `D[f, x]`.

### 14.3 Simplification

**PIRF-D-E-020** [Event-driven]
When differentiation is complete (no residual `D` operators), the Engine
shall apply algebraic simplification to the result.

**PIRF-D-E-021** [Optional feature]
Where the host CAS provides a simplifier, the Engine shall use it via the
Phrasebook's `Simplify` or `FullSimplify` implementation to produce
the simplest form of the derivative.

### 14.4 Interaction with integration

**PIRF-D-E-030** [Event-driven]
When an integration rule's constraint contains a `["D", f, x]` or
`["Dif", f, x]` expression, the Engine shall evaluate the derivative
using the differentiation rules before testing the constraint.

**PIRF-D-E-031** [Event-driven]
When an integration rule's result contains a `["D", f, x]` or
`["Dif", f, x]` expression, the Engine shall evaluate the derivative
using the differentiation rules during result instantiation.

> **Design note.** This is the primary motivation for portable
> differentiation rules: RUBI integration rules frequently invoke `D`
> and `Dif` in both constraints and results. Without portable
> differentiation, these must be delegated to CAS built-ins, breaking
> the language-independence goal.

---

## 15. Requirements — Differentiation test suite

**PIRF-D-V-001** [Ubiquitous]
The format shall include a differentiation test suite as JSON files
in a `tests-deriv/` directory, following the same schema as integration
test files (PIRF §12).

**PIRF-D-V-002** [Ubiquitous]
Each differentiation test problem shall contain: `id`, `expression`
(PIRF-Expr to differentiate), `variable` (string), `expected_derivative`
(PIRF-Expr), and optionally `order` (integer, default 1).

```json
{
  "id": 200001,
  "expression": ["Power", "x", 3],
  "variable": "x",
  "expected_derivative": ["Multiply", 3, ["Power", "x", 2]],
  "order": 1
}
```

**PIRF-D-V-003** [Event-driven]
When a test is executed, the result shall be compared to
`expected_derivative` by symbolic equivalence (not syntactic equality).

**PIRF-D-V-004** [Ubiquitous]
The differentiation test suite shall cover a minimum of **500 test
problems** spanning all sections of the differentiation taxonomy.

**PIRF-D-V-005** [Unwanted behaviour]
If the Engine produces no result within configurable timeout
(default: 10s), then the validator shall grade the test as failed.

> **Design note — Timeout.** Differentiation is algorithmically much
> simpler than integration, so the default timeout is significantly
> shorter (10s vs 120s for integration).

---

## 16. Requirements — Portability

**PIRF-D-P-001** [Ubiquitous]
All differentiation rules and test files shall be independent of any
programming language — no file shall contain executable code.

**PIRF-D-P-002** [Ubiquitous]
Differentiation rules shall use the same PIRF-Expr format, JSON
structure, and JSON Schema validation as integration rules.

**PIRF-D-P-003** [Ubiquitous]
The Phrasebook architecture (PIRF §13.3) shall apply identically to
differentiation: the Phrasebook is the only host-CAS-specific component.

**PIRF-D-P-004** [Ubiquitous]
A CAS consuming PIRF integration rules shall be able to additionally
load PIRF differentiation rules using the same Loader without
modification (except for a separate `meta.json` for the `deriv/` tree).

---

## 17. Requirements — New operators introduced

**PIRF-D-X-020** [Ubiquitous]
This extension introduces the following additional operators to the
PIRF-Expr catalogue:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `PolyGamma` | 2 | Polygamma function ψ^(n)(z), needed for Gamma derivatives |
| `Dt` | 1-2 | Total derivative |

> **Design note.** These operators are already present in Mathematica
> and most CAS systems. `PolyGamma` is essential for the derivative
> of Gamma and LogGamma. `Dt` is included for completeness but is
> used less frequently than `D` in RUBI rules.

**PIRF-D-X-021** [Ubiquitous]
The operator `PolyGamma` with arity 2 shall have the semantics:
`PolyGamma[n, z]` = ψ^(n)(z) = (d/dz)^(n+1) log(Γ(z)).
When `n = 0`, this is the digamma function.

---

## 18. Requirements — Schema extension

**PIRF-D-S-001** [Ubiquitous]
The differentiation rule file schema shall be identical to
`rule-file.schema.json` (PIRF §5.1). No new schema file is required.

**PIRF-D-S-002** [Ubiquitous]
The differentiation test file schema shall be an extension of
`test-file.schema.json` adding the optional `order` field (positive
integer, default 1).

**PIRF-D-S-003** [Ubiquitous]
The differentiation `meta.json` shall follow `meta.schema.json`
(PIRF §5.3) with section numbers in the 10.x range.

---

## 19. Traceability matrix

| Category | Identifiers | Count |
|----------|------------|-------|
| Differentiation operators (D-X) | PIRF-D-X-001 to D-X-021 | 8 |
| File format (D-F) | PIRF-D-F-001 to D-F-012 | 7 |
| Taxonomy (D-T) | PIRF-D-T-001 to D-T-002 | 2 |
| Structural rules (D-R-001–006) | PIRF-D-R-001 to D-R-006 | 6 |
| Elementary rules (D-R-010–031) | PIRF-D-R-010 to D-R-031 | 8 |
| Trig rules (D-R-040–055) | PIRF-D-R-040 to D-R-055 | 2 (tables of 6+6) |
| Hyperbolic rules (D-R-060–075) | PIRF-D-R-060 to D-R-075 | 2 (tables of 6+6) |
| Special function rules (D-R-080–115) | PIRF-D-R-080 to D-R-115 | 14 |
| Combinatorial rules (D-R-120–123) | PIRF-D-R-120 to D-R-123 | 4 |
| Higher-order rules (D-R-130–133) | PIRF-D-R-130 to D-R-133 | 4 |
| Engine (D-E) | PIRF-D-E-001 to D-E-031 | 9 |
| Test suite (D-V) | PIRF-D-V-001 to D-V-005 | 5 |
| Portability (D-P) | PIRF-D-P-001 to D-P-004 | 4 |
| Schema (D-S) | PIRF-D-S-001 to D-S-003 | 3 |
| **Total** | | **78** |

---

## 20. EARS pattern distribution

| EARS pattern | Count | % |
|-------------|-------|---|
| Ubiquitous | 59 | 76% |
| Event-driven (When) | 12 | 15% |
| Unwanted behaviour (If…then) | 4 | 5% |
| Optional feature (Where) | 2 | 3% |
| Complex (While…When) | 0 | 0% |
| State-driven (While) | 0 | 0% |

---

## Annex B — Differentiation operator mapping tables (normative)

### B.1 Differentiation operators

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML |
|-----------|-------------------|-------------------|----------------|
| `D` | *(custom)* | `calculus1#diff` | `<diff/>` |
| `Dt` | *(custom)* | `calculus1#nthdiff` (partial) | `<partialdiff/>` |
| `PolyGamma` | *(custom)* | `specfun1#polygamma` | *(none standard)* |

### B.2 New special function operators

| PIRF-Expr | Semantics | Mathematica equivalent |
|-----------|-----------|----------------------|
| `PolyGamma[0, z]` | Digamma function ψ(z) | `PolyGamma[0, z]` |
| `PolyGamma[n, z]` | Polygamma ψ^(n)(z) | `PolyGamma[n, z]` |

---

## Annex C — Differentiation rule priority order (normative)

The following loading order is **mandatory** for correct behaviour:

1. **10.1 — Structural rules** (constant, identity, linearity, sum, negation)
2. **10.2 — Elementary function rules** (power, exponential, logarithmic)
3. **10.3 — Trigonometric rules** (circular, inverse circular)
4. **10.4 — Hyperbolic rules** (hyperbolic, inverse hyperbolic)
5. **10.5 — Special function rules** (error, gamma, bessel, elliptic, etc.)
6. **10.6 — Combinatorial rules** (product, quotient, chain)
7. **10.7 — Higher-order rules** (iterated derivatives)

> **Design note — Priority.** Structural rules must precede combinatorial
> rules so that `D[Multiply[c, f], x]` with constant `c` matches the
> constant-multiple rule (§10.1) before the general product rule (§10.6).
> Similarly, specific function rules (§10.2–10.5) must precede the
> general power/exponential rules to avoid incorrect generic matching.
