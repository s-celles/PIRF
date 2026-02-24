# EARS Specification — Portable Assumption System for Symbolic Computation

**Project:** RUBI-PIRF Assumption Extension (PIRF-Assume)
**Version:** 0.1.0-draft
**Date:** 2026-02-24
**Method:** EARS (Easy Approach to Requirements Syntax) — Alistair Mavin, 2009
**System:** Any CAS implementing rule-based symbolic computation with assumptions
**Companion to:** PIRF (integration), PIRF-D (differentiation), PIRF-S (simplification), PIRF-Solve (solving)

---

## 1. Purpose and scope

### 1.1 The problem

Every PIRF specification depends on **assumptions about symbols** to
determine which rules are valid:

- PIRF-S: `Power[Power[x, a], b] → Power[x, Multiply[a, b]]` requires
  conditions on `x`, `a`, `b` (see §8 of PIRF-S).
- PIRF-S: `Log[Multiply[a, b]] → Add[Log[a], Log[b]]` requires `a, b > 0`.
- PIRF-S: `Sqrt[Power[x, 2]] → x` requires `x ≥ 0`; otherwise `→ Abs[x]`.
- PIRF-D: `D[Abs[x], x] → Sign[x]` requires `x ∈ ℝ`.
- PIRF-Solve: domain of solutions depends on whether solving over ℝ or ℂ.

Currently, PIRF has ~65 predicates (§7 of the integration spec) that
**test** properties — `IntegerQ`, `PositiveQ`, `FreeQ`, etc. — but:

1. **No mechanism to declare** that a symbol has a property.
2. **No inference engine** to propagate: if `x > 0` and `y > 0`,
   deduce `x + y > 0` and `x · y > 0`.
3. **No standard representation** for assumption sets.

This means each Phrasebook implements its own assumption system
(SymPy's `assumptions`, Mathematica's `Assumptions`/`$Assumptions`,
Julia's type annotations), leading to **semantic divergence**.

### 1.2 What this spec provides

This specification defines:

- **Property vocabulary**: a closed set of properties symbols can have.
- **Assumption declaration**: how to attach properties to symbols.
- **Inference rules**: portable rules for deducing properties of compound
  expressions from properties of their parts.
- **Assumption context**: a scoped environment carrying active assumptions.
- **Integration with predicates**: how existing PIRF predicates (§7) query
  the assumption context.

### 1.3 What this spec does NOT provide

- **A general-purpose theorem prover**: this is a lightweight property
  propagation system, not full first-order logic.
- **Cylindrical algebraic decomposition**: algorithmic; stays in Phrasebook.
- **Quantifier elimination**: algorithmic; stays in Phrasebook.
- **Interval arithmetic**: complementary technique; stays in Phrasebook.

### 1.4 Architecture

```
┌──────────────────────────────────────────────────┐
│  PIRF rules (S, D, Solve, Integration)           │
│  ─ constraints reference predicates              │
├──────────────────────────────────────────────────┤
│  PIRF-Assume inference rules (this spec)         │  ← Portable
│  ─ property propagation through operators        │
│  ─ deduction chains                              │
├──────────────────────────────────────────────────┤
│  Assumption context                              │  ← Runtime state
│  ─ symbol → property set mapping                 │
│  ─ scoped (push/pop)                             │
├──────────────────────────────────────────────────┤
│  Predicates (PIRF §7)                            │  ← Query interface
│  ─ PositiveQ, IntegerQ, etc.                     │
│  ─ now query the assumption context              │
├──────────────────────────────────────────────────┤
│  Phrasebook kernel (unchanged)                   │
│  ─ numeric evaluation as fallback                │
└──────────────────────────────────────────────────┘
```

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **PIRF-Assume** | The assumption extension specified herein |
| **Property** | A named mathematical attribute a symbol or expression may have |
| **Assumption** | A declaration that a symbol has a given property |
| **Assumption context** | An ordered set of active assumptions, scoped like a call stack |
| **Property lattice** | The partial order of properties defining implication relationships |
| **Three-valued logic** | `True`, `False`, `Unknown` — the outcome of any property query |
| **Propagation rule** | A rule deducing a property of `f(a, b, ...)` from properties of `a, b, ...` |
| **Refine** | Simplify an expression using the assumption context |

---

## 3. Requirements — Property vocabulary (§A.1)

### 3.1 Domain properties

**PIRF-Assume-P-001** [Ubiquitous]
The system shall define the following **domain properties**, organised
as a lattice from most general to most specific:

```
                    Complex
                   /       \
               Real      Imaginary
              /    \
        Rational  Irrational
        /     \
   Integer   Fractional
   /    \
Even    Odd
```

| Property | Semantics | Implies |
|----------|-----------|---------|
| `Complex` | Element of ℂ | *(top of domain lattice)* |
| `Real` | Element of ℝ | `Complex` |
| `Imaginary` | Purely imaginary (b·i, b ∈ ℝ, b ≠ 0) | `Complex` |
| `Rational` | Element of ℚ | `Real` |
| `Irrational` | Element of ℝ \ ℚ | `Real` |
| `Integer` | Element of ℤ | `Rational` |
| `Fractional` | Element of ℚ \ ℤ | `Rational` |
| `Even` | Divisible by 2 | `Integer` |
| `Odd` | Not divisible by 2 | `Integer` |
| `Prime` | Prime number | `Integer`, `Positive` |
| `Composite` | Non-prime integer > 1 | `Integer`, `Positive` |
| `Natural` | Element of ℕ = {0, 1, 2, ...} | `Integer`, `NonNegative` |

**PIRF-Assume-P-002** [Ubiquitous]
Properties in the lattice shall obey **implication**: if a symbol has
property P, it automatically has all properties implied by P.

### 3.2 Sign properties

**PIRF-Assume-P-010** [Ubiquitous]
The system shall define the following **sign properties**:

| Property | Semantics | Implies |
|----------|-----------|---------|
| `Positive` | > 0 | `NonNegative`, `NonZero`, `Real` |
| `Negative` | < 0 | `NonPositive`, `NonZero`, `Real` |
| `NonNegative` | ≥ 0 | `Real` |
| `NonPositive` | ≤ 0 | `Real` |
| `NonZero` | ≠ 0 | |
| `Zero` | = 0 | `NonNegative`, `NonPositive`, `Integer`, `Even` |

```
            Real
          /   |   \
   Positive  Zero  Negative
      \      / \      /
   NonNegative   NonPositive
          \       /
           NonZero (orthogonal: Positive ∨ Negative)
```

### 3.3 Boundedness properties

**PIRF-Assume-P-020** [Ubiquitous]
The system shall define:

| Property | Semantics |
|----------|-----------|
| `Finite` | Not ±∞ and not NaN |
| `Infinite` | Is ±∞ or ComplexInfinity |
| `Bounded` | |x| < M for some finite M |

### 3.4 Algebraic properties

**PIRF-Assume-P-030** [Ubiquitous]
The system shall define:

| Property | Semantics | Implies |
|----------|-----------|---------|
| `Algebraic` | Root of a polynomial with rational coefficients | `Complex`, `Finite` |
| `Transcendental` | Not algebraic | `Complex`, `Finite`, `Irrational` (if Real) |

### 3.5 Mutual exclusions

**PIRF-Assume-P-040** [Ubiquitous]
The system shall enforce the following **mutual exclusions** (a symbol
cannot have both properties simultaneously):

| Property A | Property B | Reason |
|-----------|-----------|--------|
| `Positive` | `Negative` | Contradiction |
| `Positive` | `Zero` | Contradiction |
| `Negative` | `Zero` | Contradiction |
| `Even` | `Odd` | Partition of integers |
| `Integer` | `Fractional` | Partition of rationals |
| `Rational` | `Irrational` | Partition of reals |
| `Real` | `Imaginary` | (except zero, which is both) |
| `Algebraic` | `Transcendental` | Partition of complex numbers |
| `Finite` | `Infinite` | Disjoint |
| `Prime` | `Composite` | Partition of integers > 1 |

**PIRF-Assume-P-041** [Unwanted behaviour]
If an assumption is declared that contradicts the existing assumption
context (via mutual exclusion or lattice implication), then the system
shall reject the assumption and emit an error.

---

## 4. Requirements — Assumption declaration (§A.2)

### 4.1 Symbol-level assumptions

**PIRF-Assume-D-001** [Ubiquitous]
Assumptions shall be declared using `["Assume", symbol, property]`:

```json
["Assume", "x", "Positive"]
["Assume", "n", "Integer"]
["Assume", "z", "Complex"]
```

**PIRF-Assume-D-002** [Ubiquitous]
Multiple properties shall be declarable on a single symbol:
`["Assume", "x", ["And", "Real", "Positive"]]`.

**PIRF-Assume-D-003** [Ubiquitous]
When a property is declared, all implied properties (§3.1, §3.2) shall
be automatically added to the symbol's property set.

### 4.2 Relational assumptions

**PIRF-Assume-D-010** [Ubiquitous]
The system shall support **relational assumptions** between symbols
and expressions:

| Form | Semantics |
|------|-----------|
| `["Assume", ["Greater", "x", 0]]` | x > 0 (equivalent to Positive) |
| `["Assume", ["Less", "x", "y"]]` | x < y |
| `["Assume", ["Element", "x", "Integers"]]` | x ∈ ℤ |
| `["Assume", ["Element", "x", ["Interval", 0, 1]]]` | x ∈ [0, 1] |
| `["Assume", ["Unequal", "x", 0]]` | x ≠ 0 |

**PIRF-Assume-D-011** [Ubiquitous]
The system shall normalise relational assumptions to property
assumptions when possible:

| Relational | Normalised to |
|-----------|---------------|
| `Greater[x, 0]` | `x` has `Positive` |
| `GreaterEqual[x, 0]` | `x` has `NonNegative` |
| `Less[x, 0]` | `x` has `Negative` |
| `Unequal[x, 0]` | `x` has `NonZero` |
| `Element[x, Integers]` | `x` has `Integer` |
| `Element[x, Reals]` | `x` has `Real` |

### 4.3 Assumption scope

**PIRF-Assume-D-020** [Ubiquitous]
Assumptions shall live in a **scoped context** that supports push/pop:

```json
["Assuming", assumptions, expr]
```

evaluates `expr` with the given assumptions active, then restores the
previous context.

**PIRF-Assume-D-021** [Ubiquitous]
The assumption context shall behave as a stack: inner `Assuming` blocks
can add assumptions but not remove outer ones. Inner assumptions take
precedence if they contradict outer ones (with a warning).

**PIRF-Assume-D-022** [Ubiquitous]
A **global assumption context** shall exist (initially empty) that is
always active. Symbols can be given permanent properties.

### 4.4 Default assumptions

**PIRF-Assume-D-030** [Ubiquitous]
By default (empty assumption context), all user-defined symbols shall
be assumed to be `Complex` — the most general domain.

> **Design note.** This matches SymPy's default and Mathematica's
> behaviour. It is the safe default: no simplification rule that
> requires `Real` or `Positive` will fire unless explicitly assumed.

**PIRF-Assume-D-031** [Ubiquitous]
Mathematical constants shall have built-in properties:

| Constant | Properties |
|----------|-----------|
| `Pi` | `Positive`, `Real`, `Transcendental`, `Irrational` |
| `E` | `Positive`, `Real`, `Transcendental`, `Irrational` |
| `I` | `Imaginary`, `NonZero`, `Algebraic` |
| `GoldenRatio` | `Positive`, `Real`, `Irrational`, `Algebraic` |
| `EulerGamma` | `Positive`, `Real` (conjectured irrational, not proven) |
| `Infinity` | `Infinite`, `Positive` |
| `NegativeInfinity` | `Infinite`, `Negative` |
| `ComplexInfinity` | `Infinite` |

### 4.5 Assumptions in rule files

**PIRF-Assume-D-040** [Ubiquitous]
PIRF rule files may include assumption declarations in their
`"assumptions"` field, specifying properties required of wildcards:

```json
{
  "id": 110064,
  "pattern": ["Power", ["Power", "x_", "a_"], "b_"],
  "constraints": [],
  "assumptions": {
    "x": "Positive"
  },
  "result": ["Power", "x", ["Multiply", "a", "b"]],
  "level": "standard"
}
```

**PIRF-Assume-D-041** [Event-driven]
When the engine evaluates a rule with an `"assumptions"` field, the
engine shall check whether the bound variables satisfy the declared
properties in the current assumption context.

**PIRF-Assume-D-042** [Event-driven]
When the `"assumptions"` field is absent, the rule shall apply
unconditionally (modulo its `constraints`).

> **Design note — assumptions vs constraints.** `constraints` are
> structural tests on the matched expression (IntegerQ, FreeQ, etc.).
> `assumptions` are property requirements on symbols that must be
> satisfied by the assumption context. A rule may have both.

---

## 5. Requirements — Three-valued query logic (§A.3)

**PIRF-Assume-Q-001** [Ubiquitous]
All property queries shall return one of three values:

| Value | Meaning |
|-------|---------|
| `True` | The property is known to hold |
| `False` | The property is known to NOT hold |
| `Unknown` | The property cannot be determined |

**PIRF-Assume-Q-002** [Ubiquitous]
Logical connectives on three-valued results shall follow
**Kleene's strong logic**:

| A | B | And(A,B) | Or(A,B) | Not(A) |
|---|---|----------|---------|--------|
| T | T | T | T | F |
| T | F | F | T | F |
| T | U | U | T | F |
| F | F | F | F | T |
| F | U | F | U | T |
| U | U | U | U | U |

**PIRF-Assume-Q-003** [Ubiquitous]
When a predicate from PIRF §7 is evaluated (e.g. `PositiveQ[x]`), it
shall query the assumption context and return `True`, `False`, or `Unknown`.

**PIRF-Assume-Q-004** [Event-driven]
When a rule constraint returns `Unknown`, the rule shall **not** be
applied. Only `True` triggers rule application.

> **Design note — Conservative semantics.** `Unknown` is treated as
> "not satisfied" for rule application. This ensures correctness at
> the expense of completeness: some valid simplifications may be missed
> when assumptions are insufficient. This is the safe default.

**PIRF-Assume-Q-005** [Optional feature]
Where the user explicitly requests aggressive simplification
(`FullSimplify`), the engine may treat `Unknown` as `True` for
non-destructive transformations (those that can be reversed), with
a warning.

---

## 6. Requirements — Inference rules (§A.4)

### 6.1 Arithmetic propagation

**PIRF-Assume-I-001** [Ubiquitous]
The system shall infer properties of **sums** from properties of operands:

| Operands | Result property |
|----------|----------------|
| `Positive + Positive` | `Positive` |
| `Positive + NonNegative` | `Positive` |
| `NonNegative + NonNegative` | `NonNegative` |
| `Negative + Negative` | `Negative` |
| `Negative + NonPositive` | `Negative` |
| `NonPositive + NonPositive` | `NonPositive` |
| `Integer + Integer` | `Integer` |
| `Rational + Rational` | `Rational` |
| `Real + Real` | `Real` |
| `Complex + Complex` | `Complex` |
| `Finite + Finite` | `Finite` |

**PIRF-Assume-I-002** [Ubiquitous]
The system shall infer properties of **products**:

| Operands | Result property |
|----------|----------------|
| `Positive × Positive` | `Positive` |
| `Positive × Negative` | `Negative` |
| `Negative × Negative` | `Positive` |
| `NonZero × NonZero` | `NonZero` |
| `Any × Zero` | `Zero` |
| `Integer × Integer` | `Integer` |
| `Rational × Rational` | `Rational` |
| `Real × Real` | `Real` |
| `Even × Integer` | `Even` |
| `Finite × Finite` | `Finite` |

**PIRF-Assume-I-003** [Ubiquitous]
The system shall infer properties of **negation**:

| Operand | Result property |
|---------|----------------|
| `Positive` | `Negative` |
| `Negative` | `Positive` |
| `NonNegative` | `NonPositive` |
| `NonPositive` | `NonNegative` |
| `Integer` | `Integer` |
| `Real` | `Real` |

**PIRF-Assume-I-004** [Ubiquitous]
The system shall infer properties of **reciprocals** (`1/x`):

| Operand | Result property |
|---------|----------------|
| `Positive` | `Positive` |
| `Negative` | `Negative` |
| `NonZero, Integer` | `Rational` |
| `NonZero, Rational` | `Rational` |
| `NonZero, Real` | `Real` |

### 6.2 Power propagation

**PIRF-Assume-I-010** [Ubiquitous]
The system shall infer properties of **powers** `x^n`:

| Base | Exponent | Result |
|------|----------|--------|
| `Positive` | `Real` | `Positive` |
| `Positive` | `Integer` | `Positive` |
| `NonNegative` | `Positive` | `NonNegative` |
| `Real, NonZero` | `Even` (integer) | `Positive` |
| `Real` | `Even` (integer) | `NonNegative` |
| `Integer` | `Natural` | `Integer` |
| `Rational` | `Integer` | `Rational` |
| `Negative` | `Odd` (integer) | `Negative` |
| `Negative` | `Even` (integer) | `Positive` |

**PIRF-Assume-I-011** [Ubiquitous]
The system shall infer properties of **Sqrt[x]**:

| Argument | Result |
|----------|--------|
| `Positive` | `Positive`, `Real` |
| `NonNegative` | `NonNegative`, `Real` |
| `Negative` | `Imaginary` (purely imaginary, positive coefficient) |
| `Zero` | `Zero` |

**PIRF-Assume-I-012** [Ubiquitous]
The system shall infer properties of **Abs[x]**:

| Argument | Result |
|----------|--------|
| `Real` | `NonNegative`, `Real` |
| `NonZero, Real` | `Positive`, `Real` |
| `Complex` | `NonNegative`, `Real` |
| `Zero` | `Zero` |

### 6.3 Function propagation

**PIRF-Assume-I-020** [Ubiquitous]
The system shall infer properties of **elementary functions**:

| Function | Argument | Result |
|----------|----------|--------|
| `Exp[x]` | `Real` | `Positive`, `Real` |
| `Exp[x]` | `Complex` | `NonZero`, `Complex` |
| `Log[x]` | `Positive` | `Real` |
| `Log[x]` | `Positive, > 1` | `Positive` |
| `Log[x]` | `Positive, < 1` | `Negative` |
| `Sin[x]` | `Real` | `Real`, `Bounded` (|sin x| ≤ 1) |
| `Cos[x]` | `Real` | `Real`, `Bounded` |
| `Tan[x]` | `Real` | `Real` |
| `Sinh[x]` | `Real` | `Real` |
| `Cosh[x]` | `Real` | `Positive`, `Real` (cosh x ≥ 1) |
| `Atan[x]` | `Real` | `Real`, `Bounded` |
| `Asin[x]` | `Real, Bounded` (|x|≤1) | `Real` |

### 6.4 Domain closure

**PIRF-Assume-I-030** [Ubiquitous]
The following **domain closure** rules shall hold — operations on
elements of a domain stay in that domain:

| Domain | Closed under |
|--------|-------------|
| `Integer` | `+`, `−`, `×` |
| `Rational` | `+`, `−`, `×`, `÷` (denominator ≠ 0) |
| `Real` | `+`, `−`, `×`, `÷`, all real-valued functions |
| `Complex` | `+`, `−`, `×`, `÷`, all standard functions |

### 6.5 Inference depth limit

**PIRF-Assume-I-040** [Ubiquitous]
Property inference shall have a configurable maximum depth (default: 10).
This prevents infinite chains when expressions are deeply nested.

**PIRF-Assume-I-041** [Event-driven]
When inference depth is exceeded, the query shall return `Unknown`.

---

## 7. Requirements — Refine operation (§A.5)

**PIRF-Assume-RF-001** [Event-driven]
When the engine receives `["Refine", expr, assumptions]`, it shall:

1. Push the given assumptions onto the assumption context.
2. Re-evaluate/simplify `expr` with the enriched context.
3. Pop the assumptions.
4. Return the refined expression.

**PIRF-Assume-RF-002** [Ubiquitous]
`Refine` shall enable simplifications that were blocked by `Unknown`
results. Examples:

| Expression | Assumptions | Refined result |
|-----------|-------------|----------------|
| `Sqrt[Power[x, 2]]` | `x: Positive` | `x` |
| `Sqrt[Power[x, 2]]` | `x: Real` | `Abs[x]` |
| `Sqrt[Power[x, 2]]` | *(none)* | `Sqrt[Power[x, 2]]` (unchanged) |
| `Log[Multiply[a, b]]` | `a: Positive, b: Positive` | `Add[Log[a], Log[b]]` |
| `Power[Power[x, a], b]` | `x: Positive` | `Power[x, Multiply[a, b]]` |
| `Abs[x]` | `x: Negative` | `Negate[x]` |
| `Sign[x]` | `x: Positive` | `1` |
| `Im[x]` | `x: Real` | `0` |
| `Re[x]` | `x: Real` | `x` |

---

## 8. Requirements — Integration with PIRF predicates (§A.6)

**PIRF-Assume-B-001** [Ubiquitous]
The existing PIRF predicates (§7) shall be **bridged** to the assumption
context as follows:

| PIRF Predicate | Assumption query |
|---------------|-----------------|
| `IntegerQ[x]` | Has property `Integer`? (or numerically evaluable to integer) |
| `PositiveQ[x]` | Has property `Positive`? |
| `NegativeQ[x]` | Has property `Negative`? |
| `NonzeroQ[x]` | Has property `NonZero`? |
| `ZeroQ[x]` | Has property `Zero`? (or numerically evaluable to 0) |
| `EvenQ[x]` | Has property `Even`? |
| `OddQ[x]` | Has property `Odd`? |
| `RealNumericQ[x]` | Has property `Real`? AND numerically evaluable |
| `FractionQ[x]` | Has property `Fractional`? |
| `PositiveIntegerQ[x]` | Has properties `Integer` AND `Positive`? |
| `NegativeIntegerQ[x]` | Has properties `Integer` AND `Negative`? |

**PIRF-Assume-B-002** [Ubiquitous]
Predicate evaluation shall follow a **two-phase** strategy:

1. **Phase 1 — Structural test**: inspect the expression syntactically
   (e.g. `3` is structurally an integer, `Divide[2, 3]` is a fraction).
2. **Phase 2 — Assumption query**: if Phase 1 is inconclusive, query
   the assumption context.

Phase 1 always takes priority (it is exact and fast).

**PIRF-Assume-B-003** [Ubiquitous]
The comparison predicates (`GtQ`, `LtQ`, `GeQ`, `LeQ`, `EqQ`, `NeQ`)
shall similarly query the assumption context when direct numeric
comparison is not possible.

**PIRF-Assume-B-004** [Event-driven]
When a predicate would return `Unknown` but the rule level is `"full"`
(FullSimplify), the engine may use the Phrasebook's numeric evaluation
as a heuristic third phase, with a configurable tolerance.

---

## 9. Requirements — Assumption file format (§A.7)

### 9.1 Assumption context in rule files

**PIRF-Assume-F-001** [Ubiquitous]
The `"assumptions"` field in rule JSON files shall have the format:

```json
{
  "assumptions": {
    "wildcard_name": "Property",
    "other_wildcard": ["And", "Property1", "Property2"]
  }
}
```

### 9.2 Assumption set files

**PIRF-Assume-F-010** [Optional feature]
Where a common set of assumptions is reused across many rules, it may
be defined in a shared file and referenced by name:

```json
// assumptions/real-positive.json
{
  "id": "real-positive",
  "description": "All variables are real and positive",
  "assumptions": {
    "x_": "Positive",
    "a_": "Positive",
    "b_": "Positive"
  }
}
```

A rule file can then reference it:
```json
{
  "id": 110230,
  "pattern": ["Log", ["Multiply", "a_", "b_"]],
  "constraints": [],
  "assumption_set": "real-positive",
  "result": ["Add", ["Log", "a"], ["Log", "b"]],
  "level": "standard"
}
```

### 9.3 Inference rules as JSON

**PIRF-Assume-F-020** [Ubiquitous]
Property inference rules (§6) shall be stored as JSON in the
`assume/` directory:

```
PIRF/
├── assume/
│   ├── meta.json
│   ├── properties.json          # Property lattice definition
│   ├── inference-arithmetic.json # Sum, product, negation rules
│   ├── inference-power.json     # Power, sqrt, abs rules
│   ├── inference-functions.json # Elementary function rules
│   └── inference-domain.json    # Domain closure rules
├── rules/
├── deriv/
├── simplify/
└── solve/
```

**PIRF-Assume-F-021** [Ubiquitous]
The property lattice shall be defined in `properties.json` with
explicit implication edges and mutual exclusion pairs.

```json
{
  "properties": {
    "Positive": {
      "implies": ["NonNegative", "NonZero", "Real"],
      "excludes": ["Negative", "Zero", "NonPositive"]
    },
    "Integer": {
      "implies": ["Rational"],
      "excludes": ["Fractional", "Irrational"]
    }
  }
}
```

---

## 10. Requirements — Assumption engine behaviour

### 10.1 Context management

**PIRF-Assume-E-001** [Ubiquitous]
The engine shall maintain an **assumption context** as a stack of
assumption frames. Each frame maps symbols to property sets.

**PIRF-Assume-E-002** [Event-driven]
When entering an `["Assuming", assumptions, expr]` block, the engine
shall push a new frame.

**PIRF-Assume-E-003** [Event-driven]
When exiting an `Assuming` block, the engine shall pop the frame,
restoring the previous context.

**PIRF-Assume-E-004** [Ubiquitous]
Property queries shall search frames top-down (most recent first).

### 10.2 Inference caching

**PIRF-Assume-E-010** [Ubiquitous]
The engine should cache inferred properties of subexpressions within
a single assumption frame to avoid redundant computation.

**PIRF-Assume-E-011** [Event-driven]
When an assumption frame is popped, its cache shall be invalidated.

### 10.3 Contradiction handling

**PIRF-Assume-E-020** [Unwanted behaviour]
If a contradiction is detected during assumption declaration (e.g.
`Assume[x, Positive]` when `x` already has `Negative`), then the
system shall reject the assumption and emit an error.

**PIRF-Assume-E-021** [Unwanted behaviour]
If a contradiction is detected during inference (e.g. propagation
yields both `Positive` and `Zero` for the same expression), then the
system shall return `Unknown` for the conflicting properties and emit
a warning.

### 10.4 Performance

**PIRF-Assume-E-030** [Ubiquitous]
Property queries for a single symbol with declared properties shall
complete in O(1) time (hash table lookup).

**PIRF-Assume-E-031** [Ubiquitous]
Property inference for compound expressions shall complete in time
proportional to expression depth, bounded by the depth limit (§6.5).

---

## 11. Requirements — Test suite

**PIRF-Assume-V-001** [Ubiquitous]
The format shall include an assumption test suite in `tests-assume/`.

**PIRF-Assume-V-002** [Ubiquitous]
Each test shall contain: `id`, `assumptions` (list of Assume
declarations), `query` (predicate call), `expected` (`True`, `False`,
or `Unknown`).

```json
{
  "id": 500001,
  "assumptions": [["Assume", "x", "Positive"]],
  "query": ["PositiveQ", ["Power", "x", 2]],
  "expected": "True"
}
```

```json
{
  "id": 500002,
  "assumptions": [["Assume", "x", "Real"]],
  "query": ["PositiveQ", ["Power", "x", 2]],
  "expected": "Unknown"
}
```

```json
{
  "id": 500003,
  "assumptions": [["Assume", "x", "Real"], ["Assume", "x", "NonZero"]],
  "query": ["PositiveQ", ["Power", "x", 2]],
  "expected": "True"
}
```

**PIRF-Assume-V-003** [Ubiquitous]
The test suite shall include **inference chain tests** verifying
multi-step deduction:

```json
{
  "id": 500010,
  "assumptions": [
    ["Assume", "x", "Positive"],
    ["Assume", "y", "Positive"]
  ],
  "query": ["PositiveQ", ["Add", "x", ["Multiply", 2, "y"]]],
  "expected": "True"
}
```

**PIRF-Assume-V-004** [Ubiquitous]
The test suite shall include **Refine tests** verifying expression
simplification under assumptions:

```json
{
  "id": 500020,
  "assumptions": [["Assume", "x", "Positive"]],
  "input": ["Sqrt", ["Power", "x", 2]],
  "expected_output": "x"
}
```

**PIRF-Assume-V-005** [Ubiquitous]
The test suite shall include a minimum of **200 test problems**:
at least 50 direct property queries, 50 inference chains, 50 Refine
tests, and 50 contradiction/edge cases.

**PIRF-Assume-V-006** [Ubiquitous]
Test timeout shall be 1 second (assumption queries must be fast).

---

## 12. Traceability matrix

| Category | Identifiers | Count |
|----------|------------|-------|
| Properties — domain (P-001–002) | PIRF-Assume-P-001 to P-002 | 2 |
| Properties — sign (P-010) | PIRF-Assume-P-010 | 1 |
| Properties — boundedness (P-020) | PIRF-Assume-P-020 | 1 |
| Properties — algebraic (P-030) | PIRF-Assume-P-030 | 1 |
| Properties — exclusion (P-040–041) | PIRF-Assume-P-040 to P-041 | 2 |
| Declaration (D-001–042) | PIRF-Assume-D-001 to D-042 | 13 |
| Three-valued logic (Q-001–005) | PIRF-Assume-Q-001 to Q-005 | 5 |
| Inference — arithmetic (I-001–004) | PIRF-Assume-I-001 to I-004 | 4 |
| Inference — power (I-010–012) | PIRF-Assume-I-010 to I-012 | 3 |
| Inference — functions (I-020) | PIRF-Assume-I-020 | 1 |
| Inference — domain closure (I-030) | PIRF-Assume-I-030 | 1 |
| Inference — limits (I-040–041) | PIRF-Assume-I-040 to I-041 | 2 |
| Refine (RF-001–002) | PIRF-Assume-RF-001 to RF-002 | 2 |
| Bridge to predicates (B-001–004) | PIRF-Assume-B-001 to B-004 | 4 |
| File format (F-001–021) | PIRF-Assume-F-001 to F-021 | 4 |
| Engine (E-001–031) | PIRF-Assume-E-001 to E-031 | 9 |
| Test suite (V-001–006) | PIRF-Assume-V-001 to V-006 | 6 |
| **Total** | | **61** |

---

## 13. EARS pattern distribution

| EARS pattern | Count | % |
|-------------|-------|---|
| Ubiquitous | 42 | 69% |
| Event-driven (When) | 11 | 18% |
| Unwanted behaviour (If…then) | 4 | 7% |
| Optional feature (Where) | 4 | 7% |

---

## Annex H — Comparison with existing assumption systems

| Feature | Mathematica | SymPy | Julia/Symbolics | **PIRF-Assume** |
|---------|------------|-------|-----------------|-----------------|
| Declaration | `Assuming[x>0, ...]` | `Symbol('x', positive=True)` | `@syms x::Real` | `["Assume", "x", "Positive"]` |
| Storage | Global `$Assumptions` | Per-symbol at creation | Type parameter | Scoped context stack |
| Inference | Yes (built into kernel) | Yes (ask/refine system) | Minimal (type propagation) | Portable JSON rules |
| Three-valued | Yes | Yes (T/F/None) | Partial | Yes (T/F/Unknown) |
| Portable | No | No (Python only) | No (Julia only) | **Yes (JSON)** |
| Scope | `Assuming` block | Global or `with assuming` | Lexical (type) | `Assuming` block + global |
| Contradiction detection | Yes | Yes | No | Yes |
| Property lattice | Implicit | Implicit | None | **Explicit JSON definition** |

---

## Annex I — Updated dependency diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PIRF-Expr (shared language)                      │
├─────────────────────────────────────────────────────────────────────┤
│  PIRF-Assume (§A)                                                    │
│  ─ Property lattice, inference rules, assumption context             │
│  ─ FOUNDATION: queried by all other specs                            │
├─────────────────────────────────────────────────────────────────────┤
│  PIRF-S (§11)  │  PIRF-D (§10)  │  PIRF-Solve (§12)  │  PIRF (§1–9)│
│  Simplification│  Derivation    │  Solving           │  Integration │
│  ~225 rules    │  ~100 rules    │  ~126 rules        │  ~7,800 rules│
├──────────────────────────────────────────────────────────────────────┤
│  Phrasebook kernel: GCD, factorisation, Gaussian elim., numeric eval │
├──────────────────────────────────────────────────────────────────────┤
│  Host CAS: Julia / Python / Rust / JS / Mathematica / ...            │
└──────────────────────────────────────────────────────────────────────┘

PIRF-Assume is queried by:
  ├── PIRF-S  (branch cut conditions, identity validity)
  ├── PIRF-D  (domain of derivative rules)
  ├── PIRF-Solve (solution domain, existence)
  └── PIRF    (constraint evaluation via predicates)
```
