# EARS Specification — Portable Format for Symbolic Equation Solving Rules

**Project:** RUBI-PIRF Equation Solving Extension (PIRF-Solve)
**Version:** 0.1.0-draft
**Date:** 2026-02-24
**Method:** EARS (Easy Approach to Requirements Syntax) — Alistair Mavin, 2009
**System:** Any CAS implementing rule-based symbolic equation solving
**Companion to:** PIRF v0.1.0-draft (integration), PIRF-D (differentiation), PIRF-S (simplification)

---

## 1. Purpose and scope

This document specifies the requirements for a **portable rule set for
symbolic equation solving**, expressed in the PIRF JSON format. Equation
solving is the second most-requested CAS operation after simplification,
and is a prerequisite for many higher-level operations (limits, series,
integration by partial fractions, etc.).

### 1.1 Why equation solving rules must be portable

Currently, every CAS implements its own `Solve` with different:

- **Coverage**: some handle transcendental equations, others don't.
- **Solution forms**: SymPy returns sets, Mathematica returns rules,
  Julia/Symbolics.jl returns arrays.
- **Branch handling**: which roots are returned for multi-valued equations.

A portable rule set ensures consistent solving behaviour across all
PIRF-consuming CAS implementations.

### 1.2 Scope

This specification covers:

- **Linear equations** (single variable, systems)
- **Polynomial equations** (quadratic, cubic, quartic formulas; numeric
  roots for degree ≥ 5)
- **Rational equations** (clearing denominators)
- **Radical equations** (isolation and squaring)
- **Exponential equations** (logarithmic transformation)
- **Logarithmic equations** (exponentiation)
- **Trigonometric equations** (standard forms, periodic solutions)
- **Systems of equations** (linear systems, substitution, elimination)
- **Inequalities** (linear, polynomial, rational)
- **Ordinary differential equations** (separable, linear 1st/2nd order,
  exact, Bernoulli, Riccati — rule-based classification and solution)

This specification does **not** cover:

- **Partial differential equations** — future extension.
- **Diophantine equations** — number-theoretic, largely algorithmic.
- **Numerical root-finding** (Newton, bisection) — Phrasebook algorithm.
- **Gröbner basis methods** for polynomial systems — Phrasebook algorithm.

### 1.3 Relationship to other PIRF specifications

- **Depends on PIRF-S** (simplification): solutions must be simplified.
- **Depends on PIRF-D** (differentiation): ODE solving requires derivatives.
- **Used by PIRF** (integration): partial fraction decomposition requires
  root finding; some substitution rules require solving auxiliary equations.

### 1.4 Architecture: rules vs algorithms

As with PIRF-S, equation solving is a hybrid:

```
┌──────────────────────────────────────────┐
│  Solving rules (PIRF-Solve)              │  ← This spec: portable JSON rules
│  ─ classification, formula application,  │
│    transformation strategies             │
├──────────────────────────────────────────┤
│  Algorithmic kernel (Phrasebook)         │  ← Host-CAS-specific
│  ─ polynomial GCD, resultant,            │
│    Gröbner basis, numeric root-finding,  │
│    linear algebra (Gaussian elimination) │
├──────────────────────────────────────────┤
│  PIRF-S (simplification)                 │  ← Portable simplification
├──────────────────────────────────────────┤
│  PIRF-Expr (unchanged)                   │  ← Shared expression format
└──────────────────────────────────────────┘
```

---

## 2. Glossary (additions to PIRF §2)

| Term | Definition |
|------|-----------|
| **PIRF-Solve** | PIRF Equation Solving extension — the extension specified herein |
| **Equation** | An expression of the form `Equal[lhs, rhs]` or equivalently `Equal[expr, 0]` |
| **Solution** | A value or set of values for the unknowns that satisfies the equation |
| **Solution set** | The complete set of solutions, possibly parameterised |
| **Extraneous solution** | A value introduced by a non-reversible transformation (squaring, clearing denominators) that does not satisfy the original equation |
| **Principal solution** | A solution in the principal branch/period |
| **General solution** | A solution parameterised by an integer (for periodic) or constant (for ODE) |
| **ODE** | Ordinary Differential Equation |
| **IVP** | Initial Value Problem — ODE with initial condition(s) |
| **BVP** | Boundary Value Problem — ODE with boundary condition(s) |

---

## 3. EARS patterns used

*(Same as PIRF §3.)*

---

## 4. Requirements — Solving-specific operators

### 4.1 Core operators

**PIRF-Solve-X-001** [Ubiquitous]
PIRF-Expr shall define the following **equation and solving operators**:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `Equal` | 2 | Equation: $1 = $2 |
| `Unequal` | 2 | Disequation: $1 ≠ $2 |
| `Less` | 2 | Strict inequality: $1 < $2 |
| `Greater` | 2 | Strict inequality: $1 > $2 |
| `LessEqual` | 2 | Inequality: $1 ≤ $2 |
| `GreaterEqual` | 2 | Inequality: $1 ≥ $2 |
| `Solve` | 2-3 | Solve equation(s) $1 for variable(s) $2, optionally over domain $3 |
| `DSolve` | 3 | Solve ODE $1 for function $2 w.r.t. variable $3 |
| `Reduce` | 2-3 | Reduce system to explicit conditions on $2, over domain $3 |
| `Roots` | 2 | Polynomial roots of $1 in variable $2 |
| `NSolve` | 2 | Numeric solve of $1 for $2 |

**PIRF-Solve-X-002** [Ubiquitous]
PIRF-Expr shall define the following **solution representation operators**:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `Solution` | 1+ | An ordered set of solutions: `Solution[s1, s2, ...]` |
| `ConditionalSolution` | 2 | Solution $1 valid under condition $2 |
| `ParametricSolution` | 2 | Solution $1 parameterised by $2 (e.g. integer parameter) |
| `NoSolution` | 0 | Empty solution set (equation has no solution) |
| `AllSolutions` | 0 | Universal: every value is a solution (identity equation) |
| `Interval` | 2 | Closed interval [$1, $2] |
| `OpenInterval` | 2 | Open interval ($1, $2) |
| `Union` | 2+ | Set union of solution sets |
| `Intersection` | 2+ | Set intersection |
| `Element` | 2 | $1 belongs to domain $2 |
| `Integers` | 0 | The set ℤ |
| `Rationals` | 0 | The set ℚ |
| `Reals` | 0 | The set ℝ |
| `Complexes` | 0 | The set ℂ |
| `PositiveReals` | 0 | The set ℝ⁺ |
| `C` | 1 | Arbitrary constant C(n) for ODE general solutions |

> **Design note — Solution representation.** Mathematica uses replacement
> rules `{x -> 3}`, SymPy uses `FiniteSet`, Julia uses arrays. PIRF-Solve
> uses a neutral `Solution[...]` wrapper. The Phrasebook translates to the
> host CAS's native representation.

### 4.2 Classification predicates

**PIRF-Solve-X-010** [Ubiquitous]
PIRF-Expr shall define the following **equation classification predicates**:

| Predicate | Arity | Semantics |
|-----------|-------|-----------|
| `LinearEquationQ` | 2 | Equation is linear in variable |
| `QuadraticEquationQ` | 2 | Equation is quadratic in variable |
| `CubicEquationQ` | 2 | Equation is cubic in variable |
| `QuarticEquationQ` | 2 | Equation is quartic in variable |
| `PolynomialEquationQ` | 2 | Equation is polynomial in variable |
| `RationalEquationQ` | 2 | Equation is rational in variable |
| `RadicalEquationQ` | 2 | Equation contains radicals of variable |
| `ExponentialEquationQ` | 2 | Equation involves exponentials of variable |
| `LogarithmicEquationQ` | 2 | Equation involves logarithms of variable |
| `TrigonometricEquationQ` | 2 | Equation involves trig functions of variable |
| `LinearSystemQ` | 2 | System is linear in variables |
| `SeparableODEQ` | 3 | ODE is separable |
| `LinearODEQ` | 3 | ODE is linear |
| `ExactODEQ` | 3 | ODE is exact |
| `BernoulliODEQ` | 3 | ODE is Bernoulli type |
| `RiccatiODEQ` | 3 | ODE is Riccati type |
| `HomogeneousODEQ` | 3 | ODE is homogeneous |
| `SecondOrderLinearODEQ` | 3 | ODE is 2nd-order linear |

---

## 5. Requirements — Equation normalisation (§12.1)

**PIRF-Solve-R-001** [Event-driven]
When the engine receives `["Solve", eq, x]`, the engine shall first
normalise the equation to the form `["Equal", expr, 0]` by subtracting
the right-hand side from the left-hand side.

```json
{
  "id": 120001,
  "pattern": ["Solve", ["Equal", "lhs_", "rhs_"], "x_"],
  "constraints": [["Not", ["ZeroQ", "rhs"]]],
  "result": ["Solve", ["Equal", ["Subtract", "lhs", "rhs"], 0], "x"]
}
```

**PIRF-Solve-R-002** [Event-driven]
When the engine receives `["Solve", expr, x]` where `expr` is not
wrapped in `Equal`, the engine shall interpret it as
`["Solve", ["Equal", expr, 0], x]`.

**PIRF-Solve-R-003** [Event-driven]
When the normalised equation is `["Equal", 0, 0]`, the engine shall
return `AllSolutions`.

**PIRF-Solve-R-004** [Event-driven]
When the normalised equation contains no occurrence of the variable `x`,
and is not identically zero, the engine shall return `NoSolution`.

---

## 6. Requirements — Linear equation rules (§12.2)

**PIRF-Solve-R-010** [Event-driven]
When the equation is linear in `x` (form `a·x + b = 0` with `a ≠ 0`),
the engine shall return `Solution[Divide[Negate[b], a]]`.

```json
{
  "id": 120010,
  "pattern": ["Solve", ["Equal", ["Add", ["Multiply", "a_", "x_"], "b_"], 0], "x_"],
  "constraints": [
    ["FreeQ", "a", "x"],
    ["FreeQ", "b", "x"],
    ["NonzeroQ", "a"]
  ],
  "result": ["Solution", ["Divide", ["Negate", "b"], "a"]]
}
```

**PIRF-Solve-R-011** [Unwanted behaviour]
If the equation is linear and `a = 0` and `b ≠ 0`, then the engine
shall return `NoSolution`.

**PIRF-Solve-R-012** [Unwanted behaviour]
If the equation is linear and `a = 0` and `b = 0`, then the engine
shall return `AllSolutions`.

---

## 7. Requirements — Polynomial equation rules (§12.3)

### 7.1 Quadratic equations

**PIRF-Solve-R-020** [Event-driven]
When the equation is quadratic (`a·x² + b·x + c = 0` with `a ≠ 0`),
the engine shall compute the discriminant `Δ = b² − 4ac` and return:

| Condition | Result |
|-----------|--------|
| Δ > 0 (real) | `Solution[Divide[Add[Negate[b], Sqrt[Δ]], Multiply[2, a]], Divide[Subtract[Negate[b], Sqrt[Δ]], Multiply[2, a]]]` |
| Δ = 0 | `Solution[Divide[Negate[b], Multiply[2, a]]]` (double root) |
| Δ < 0 (over ℝ) | `NoSolution` |
| Δ < 0 (over ℂ) | Two complex conjugate solutions using `I` |

```json
{
  "id": 120020,
  "pattern": ["Solve", ["Equal", ["Add", ["Multiply", "a_", ["Power", "x_", 2]], ["Multiply", "b_", "x_"], "c_"], 0], "x_"],
  "constraints": [
    ["FreeQ", ["List", "a", "b", "c"], "x"],
    ["NonzeroQ", "a"]
  ],
  "result": ["With",
    ["List", ["Rule", "disc", ["Subtract", ["Power", "b", 2], ["Multiply", 4, "a", "c"]]]],
    ["Solution",
      ["Divide", ["Add", ["Negate", "b"], ["Sqrt", "disc"]], ["Multiply", 2, "a"]],
      ["Divide", ["Subtract", ["Negate", "b"], ["Sqrt", "disc"]], ["Multiply", 2, "a"]]
    ]
  ]
}
```

**PIRF-Solve-R-021** [Optional feature]
Where solving over ℝ is requested (domain argument = `Reals`), the engine
shall discard complex solutions.

### 7.2 Cubic equations

**PIRF-Solve-R-030** [Event-driven]
When the equation is cubic (`a·x³ + b·x² + c·x + d = 0`) and the
depressed cubic substitution `x = t − b/(3a)` yields `t³ + pt + q = 0`,
the engine shall apply **Cardano's formula**:

```
t = CubeRoot[−q/2 + Sqrt[q²/4 + p³/27]] + CubeRoot[−q/2 − Sqrt[q²/4 + p³/27]]
```

**PIRF-Solve-R-031** [Ubiquitous]
The engine shall return all three roots (real and/or complex) using
the appropriate branch of Cardano's formula or the trigonometric form
when the discriminant is negative (casus irreducibilis).

> **Design note — CubeRoot operator.** This specification uses `Rt[x, 3]`
> (PIRF's existing n-th root operator) for cube roots, choosing the real
> root when available.

### 7.3 Quartic equations

**PIRF-Solve-R-040** [Event-driven]
When the equation is quartic (`a·x⁴ + b·x³ + c·x² + d·x + e = 0`),
the engine shall apply **Ferrari's method**: reduce to a resolvent cubic,
solve it, then factor the quartic into two quadratics.

**PIRF-Solve-R-041** [Ubiquitous]
The engine shall return all four roots.

### 7.4 Higher-degree polynomials

**PIRF-Solve-R-050** [Event-driven]
When the equation is polynomial of degree ≥ 5, the engine shall attempt:

1. **Rational root theorem**: test `±(factors of constant) / (factors of leading coeff)`
2. **Factorisation**: delegate to Phrasebook's `Factor` to find reducible factors
3. **Special forms**: detect biquadratic (even powers only), reciprocal polynomials, etc.

**PIRF-Solve-R-051** [Unwanted behaviour]
If no closed-form solution is found for degree ≥ 5, the engine shall
return the roots in implicit form `["Roots", poly, x]` and, where
`NSolve` is available, suggest numeric solving.

> **Design note — Abel-Ruffini.** The general polynomial of degree ≥ 5
> has no solution in radicals. The engine does not attempt the impossible
> but handles special solvable cases.

### 7.5 Factored and product-zero forms

**PIRF-Solve-R-055** [Event-driven]
When the equation has the form `Multiply[f1, f2, ...] = 0`, the engine
shall solve each factor independently:
`Solve[Multiply[f1, f2], x]` → `Union[Solve[f1, x], Solve[f2, x]]`.

```json
{
  "id": 120055,
  "pattern": ["Solve", ["Equal", ["Multiply", "f_", "g_"], 0], "x_"],
  "constraints": [],
  "result": ["Union", ["Solve", ["Equal", "f", 0], "x"], ["Solve", ["Equal", "g", 0], "x"]]
}
```

**PIRF-Solve-R-056** [Event-driven]
When the equation has the form `Power[f, n] = 0` with `n > 0`, the
engine shall solve `f = 0`.

---

## 8. Requirements — Rational equation rules (§12.4)

**PIRF-Solve-R-060** [Event-driven]
When the equation is rational (contains `Divide` with `x` in denominators),
the engine shall:

1. Identify all denominators containing `x`.
2. Compute the common denominator.
3. Multiply both sides by the common denominator.
4. Solve the resulting polynomial equation.
5. **Check for extraneous solutions** (§8.1).

**PIRF-Solve-R-061** [Ubiquitous]
After solving, the engine shall verify each candidate solution does not
make any original denominator zero.

**PIRF-Solve-R-062** [Event-driven]
When a candidate solution makes a denominator zero, the engine shall
discard it from the solution set.

---

## 9. Requirements — Radical equation rules (§12.5)

**PIRF-Solve-R-070** [Event-driven]
When the equation contains radicals (fractional powers) of the variable,
the engine shall:

1. Isolate a radical on one side.
2. Raise both sides to the appropriate power to eliminate the radical.
3. Repeat until all radicals are eliminated.
4. Solve the resulting equation.
5. **Check for extraneous solutions.**

```json
{
  "id": 120070,
  "pattern": ["Solve", ["Equal", ["Sqrt", "f_"], "g_"], "x_"],
  "constraints": [["Not", ["FreeQ", "f", "x"]]],
  "result": ["Solve",
    ["And",
      ["Equal", "f", ["Power", "g", 2]],
      ["GreaterEqual", "g", 0]
    ],
    "x"
  ]
}
```

**PIRF-Solve-R-071** [Ubiquitous]
After squaring (or raising to any even power), the engine shall verify
each candidate against the original equation. Extraneous solutions shall
be discarded.

**PIRF-Solve-R-072** [Ubiquitous]
The engine shall track the **squaring depth** and enforce a maximum
(default: 4) to prevent combinatorial explosion from nested radicals.

**PIRF-Solve-R-073** [Unwanted behaviour]
If the squaring depth limit is exceeded, the engine shall return the
equation in unevaluated form `["Solve", eq, x]`.

---

## 10. Requirements — Exponential equation rules (§12.6)

**PIRF-Solve-R-080** [Event-driven]
When the equation has the form `Power[a, f(x)] = b` with `a` free of `x`
and `a > 0`, the engine shall apply logarithms:
`f(x) = Divide[Log[b], Log[a]]`.

```json
{
  "id": 120080,
  "pattern": ["Solve", ["Equal", ["Power", "a_", "f_"], "b_"], "x_"],
  "constraints": [
    ["FreeQ", "a", "x"],
    ["FreeQ", "b", "x"],
    ["PositiveQ", "a"],
    ["PositiveQ", "b"],
    ["Not", ["FreeQ", "f", "x"]]
  ],
  "result": ["Solve", ["Equal", "f", ["Divide", ["Log", "b"], ["Log", "a"]]], "x"]
}
```

**PIRF-Solve-R-081** [Event-driven]
When the equation has the form `Exp[f(x)] = b`, the engine shall apply:
`f(x) = Log[b]` (requires `b > 0`).

**PIRF-Solve-R-082** [Event-driven]
When the equation is of the form `a^f(x) + b·a^g(x) + c = 0`
(exponential quadratic), the engine shall substitute `u = a^(common factor)`
and solve the resulting quadratic.

**PIRF-Solve-R-083** [Unwanted behaviour]
If `b ≤ 0` in `a^f = b`, the engine shall return `NoSolution` over ℝ,
or complex solutions over ℂ where `Log[b] = Log[Abs[b]] + I·Pi`.

---

## 11. Requirements — Logarithmic equation rules (§12.7)

**PIRF-Solve-R-090** [Event-driven]
When the equation has the form `Log[f(x)] = c` with `c` free of `x`,
the engine shall exponentiate: `f(x) = Exp[c]`.

**PIRF-Solve-R-091** [Event-driven]
When the equation involves sums of logarithms, the engine shall apply
logarithmic identities (PIRF-S §12.2) to combine them before solving:
`Log[f] + Log[g] = c` → `Log[Multiply[f, g]] = c` → `Multiply[f, g] = Exp[c]`.

**PIRF-Solve-R-092** [Ubiquitous]
After solving, the engine shall verify that all logarithm arguments
are positive in the candidate solutions. Solutions yielding non-positive
arguments shall be discarded.

---

## 12. Requirements — Trigonometric equation rules (§12.8)

### 12.1 Basic forms

**PIRF-Solve-R-100** [Event-driven]
When the equation has a basic trigonometric form, the engine shall
return principal and general solutions:

| ID | Equation | Principal solution | General solution |
|----|----------|--------------------|-----------------|
| 120100 | `Sin[f] = a` | `f = Asin[a]` | `f = Add[Asin[a], Multiply[2, Pi, "n"]]` and `f = Subtract[Pi, Add[Asin[a], Multiply[2, Pi, "n"]]]` |
| 120101 | `Cos[f] = a` | `f = Acos[a]` | `f = Add[Acos[a], Multiply[2, Pi, "n"]]` and `f = Negate[Add[Acos[a], Multiply[2, Pi, "n"]]]` |
| 120102 | `Tan[f] = a` | `f = Atan[a]` | `f = Add[Atan[a], Multiply[Pi, "n"]]` |

where `n` is a `ParametricSolution` parameter with `Element[n, Integers]`.

**PIRF-Solve-R-101** [Event-driven]
When `|a| > 1` in `Sin[f] = a` or `Cos[f] = a`, the engine shall
return `NoSolution` over ℝ.

### 12.2 Quadratic-in-trig forms

**PIRF-Solve-R-110** [Event-driven]
When the equation has the form
`a·Sin[x]² + b·Sin[x] + c = 0` (or analogous for Cos, Tan), the engine
shall substitute `u = Sin[x]`, solve the quadratic in `u`, then solve
`Sin[x] = u_i` for each root.

### 12.3 Linear combinations

**PIRF-Solve-R-115** [Event-driven]
When the equation has the form `a·Sin[x] + b·Cos[x] = c`, the engine
shall apply the auxiliary angle method:
`R·Sin[x + φ] = c` where `R = Sqrt[a² + b²]` and `φ = Atan[b, a]`.

### 12.4 Domain control

**PIRF-Solve-R-120** [Optional feature]
Where the user specifies a domain restriction (e.g. `x ∈ [0, 2π)`),
the engine shall filter general solutions to those within the domain.

**PIRF-Solve-R-121** [Ubiquitous]
By default, `Solve` shall return **principal solutions** only.
`Reduce` shall return **general solutions** with integer parameters.

---

## 13. Requirements — Systems of equations (§12.9)

### 13.1 Linear systems

**PIRF-Solve-R-130** [Event-driven]
When the input is a system of linear equations in multiple variables,
the engine shall:

1. Express the system as `["Equal", ["Multiply", A, x_vec], b_vec]`.
2. Delegate to the Phrasebook's Gaussian elimination / LU decomposition.
3. Return solutions or indicate the system is inconsistent / underdetermined.

**PIRF-Solve-R-131** [Event-driven]
When the linear system has a unique solution, the engine shall return
`Solution[Rule[x1, v1], Rule[x2, v2], ...]`.

**PIRF-Solve-R-132** [Event-driven]
When the linear system is underdetermined (infinite solutions), the
engine shall return a `ParametricSolution` with free variables as
parameters.

**PIRF-Solve-R-133** [Event-driven]
When the linear system is inconsistent (no solution), the engine shall
return `NoSolution`.

### 13.2 Nonlinear systems

**PIRF-Solve-R-140** [Event-driven]
When the input is a system of nonlinear equations, the engine shall
attempt the following strategies in order:

1. **Substitution**: solve one equation for one variable, substitute into others.
2. **Elimination**: use resultants to eliminate variables (Phrasebook algorithm).
3. **Symmetry exploitation**: detect symmetric systems and reduce.

**PIRF-Solve-R-141** [Unwanted behaviour]
If no strategy succeeds for a nonlinear system, the engine shall return
the system in unevaluated form and suggest `NSolve` for numeric solutions.

---

## 14. Requirements — Inequality rules (§12.10)

### 14.1 Linear inequalities

**PIRF-Solve-R-150** [Event-driven]
When the inequality is linear (`a·x + b > 0` with `a ≠ 0`), the engine
shall return:

| Condition | Result |
|-----------|--------|
| a > 0 | `Solution[OpenInterval[Divide[Negate[b], a], Infinity]]` |
| a < 0 | `Solution[OpenInterval[Negate[Infinity], Divide[Negate[b], a]]]` |

**PIRF-Solve-R-151** [Ubiquitous]
The engine shall handle all four inequality operators (`Less`, `Greater`,
`LessEqual`, `GreaterEqual`) and return appropriate open or closed intervals.

### 14.2 Polynomial inequalities

**PIRF-Solve-R-160** [Event-driven]
When the inequality is polynomial, the engine shall:

1. Find the roots of the polynomial (using §7 rules).
2. Determine the sign of the polynomial on each interval between roots.
3. Return the union of intervals satisfying the inequality.

### 14.3 Rational inequalities

**PIRF-Solve-R-165** [Event-driven]
When the inequality involves a rational expression, the engine shall:

1. Move everything to one side to get `f(x)/g(x) > 0` (or similar).
2. Find roots of both numerator and denominator.
3. Build a sign chart.
4. Return the union of intervals satisfying the inequality.

**PIRF-Solve-R-166** [Ubiquitous]
Points where the denominator is zero shall be excluded from the
solution set (open intervals at those points).

### 14.4 Absolute value inequalities

**PIRF-Solve-R-170** [Event-driven]
When the inequality involves `Abs`, the engine shall apply:

| ID | Rule |
|----|------|
| 120170 | `Less[Abs[f], a]` (a > 0) → `And[Less[Negate[a], f], Less[f, a]]` |
| 120171 | `Greater[Abs[f], a]` (a > 0) → `Or[Less[f, Negate[a]], Greater[f, a]]` |
| 120172 | `Less[Abs[f], a]` (a ≤ 0) → `NoSolution` |

---

## 15. Requirements — Ordinary differential equation rules (§12.11)

### 15.1 ODE classification

**PIRF-Solve-R-200** [Event-driven]
When the engine receives `["DSolve", ode, y, x]`, the engine shall
first classify the ODE by testing classification predicates in the
following priority order:

1. `SeparableODEQ`
2. `LinearODEQ` (first-order)
3. `ExactODEQ`
4. `BernoulliODEQ`
5. `RiccatiODEQ`
6. `HomogeneousODEQ`
7. `SecondOrderLinearODEQ`

### 15.2 Separable ODEs

**PIRF-Solve-R-210** [Event-driven]
When the ODE is separable (dy/dx = f(x)·g(y)), the engine shall
separate variables and integrate both sides:
`∫ dy/g(y) = ∫ f(x) dx + C(1)`.

```json
{
  "id": 120210,
  "pattern": ["DSolve",
    ["Equal", ["D", "y_", "x_"], ["Multiply", "fx_", "gy_"]],
    "y_", "x_"],
  "constraints": [
    ["FreeQ", "fx", "y"],
    ["FreeQ", "gy", "x"]
  ],
  "result": ["Solve",
    ["Equal",
      ["Int", ["Divide", 1, "gy"], "y"],
      ["Add", ["Int", "fx", "x"], ["C", 1]]
    ],
    "y"
  ]
}
```

### 15.3 First-order linear ODEs

**PIRF-Solve-R-220** [Event-driven]
When the ODE is first-order linear (dy/dx + P(x)·y = Q(x)), the
engine shall apply the **integrating factor** method:

```
μ(x) = Exp[∫ P(x) dx]
y = Divide[∫ μ(x)·Q(x) dx + C(1), μ(x)]
```

**PIRF-Solve-R-221** [Ubiquitous]
The integrating factor shall be computed using PIRF integration rules.

### 15.4 Exact ODEs

**PIRF-Solve-R-230** [Event-driven]
When the ODE `M(x,y)dx + N(x,y)dy = 0` satisfies `∂M/∂y = ∂N/∂x`
(exact condition), the engine shall find F(x,y) such that
`∂F/∂x = M` and `∂F/∂y = N`, and return `Equal[F(x,y), C(1)]`.

**PIRF-Solve-R-231** [Ubiquitous]
The exactness test shall use PIRF-D differentiation rules to compute
the partial derivatives.

### 15.5 Bernoulli ODEs

**PIRF-Solve-R-240** [Event-driven]
When the ODE has the form `dy/dx + P(x)·y = Q(x)·y^n` (n ≠ 0, 1),
the engine shall substitute `v = y^(1−n)` to reduce to a first-order
linear ODE.

### 15.6 Second-order linear ODEs with constant coefficients

**PIRF-Solve-R-250** [Event-driven]
When the ODE has the form `a·y'' + b·y' + c·y = 0` (homogeneous,
constant coefficients), the engine shall:

1. Form the characteristic equation `a·r² + b·r + c = 0`.
2. Solve for `r` using the quadratic formula.
3. Return the general solution based on the discriminant:

| Discriminant | Solution |
|-------------|----------|
| Δ > 0 (real distinct roots r₁, r₂) | `y = Multiply[C(1), Exp[Multiply[r₁, x]]] + Multiply[C(2), Exp[Multiply[r₂, x]]]` |
| Δ = 0 (repeated root r) | `y = Multiply[Add[C(1), Multiply[C(2), x]], Exp[Multiply[r, x]]]` |
| Δ < 0 (complex roots α ± βi) | `y = Multiply[Exp[Multiply[α, x]], Add[Multiply[C(1), Cos[Multiply[β, x]]], Multiply[C(2), Sin[Multiply[β, x]]]]]` |

### 15.7 Non-homogeneous second-order (variation of parameters)

**PIRF-Solve-R-260** [Event-driven]
When the ODE has the form `a·y'' + b·y' + c·y = f(x)` (non-homogeneous),
the engine shall:

1. Solve the homogeneous part (§15.6).
2. Apply **variation of parameters** or **undetermined coefficients** to
   find a particular solution.
3. Return `y = y_h + y_p`.

**PIRF-Solve-R-261** [Event-driven]
When `f(x)` is a polynomial, exponential, sine, cosine, or product thereof,
the engine shall apply the **method of undetermined coefficients**.

**PIRF-Solve-R-262** [Optional feature]
Where `f(x)` is not amenable to undetermined coefficients, the engine
shall apply **variation of parameters** using PIRF integration rules.

### 15.8 Initial and boundary value problems

**PIRF-Solve-R-270** [Event-driven]
When initial conditions are provided (e.g. `y(0) = 1`, `y'(0) = 0`),
the engine shall substitute them into the general solution and solve
for the arbitrary constants `C(1)`, `C(2)`, etc.

**PIRF-Solve-R-271** [Ubiquitous]
The engine shall verify that the number of initial/boundary conditions
matches the order of the ODE.

**PIRF-Solve-R-272** [Unwanted behaviour]
If the conditions are insufficient (underdetermined) or inconsistent,
the engine shall return the partially determined solution or `NoSolution`
respectively.

---

## 16. Requirements — Extraneous solution checking (§12.12)

**PIRF-Solve-R-300** [Ubiquitous]
The engine shall **track all non-reversible transformations** applied
during solving (squaring, multiplying by expressions containing the
variable, clearing denominators).

**PIRF-Solve-R-301** [Event-driven]
When at least one non-reversible transformation was applied, the engine
shall verify each candidate solution by substitution into the original
equation.

**PIRF-Solve-R-302** [Event-driven]
When a candidate solution fails verification, the engine shall discard
it and annotate the solution set with `"extraneous_removed": true`.

**PIRF-Solve-R-303** [Ubiquitous]
Verification shall use PIRF-S simplification to reduce the substituted
expression and check equivalence to zero (or the appropriate truth value
for inequalities).

---

## 17. Requirements — Solving engine

### 17.1 Strategy dispatch

**PIRF-Solve-E-001** [Event-driven]
When the engine receives a `Solve` call, the engine shall classify
the equation using the predicates in §4.2 and dispatch to the
appropriate rule section.

**PIRF-Solve-E-002** [Ubiquitous]
Classification shall follow the priority order:

1. Trivial (constant, identity) — §5
2. Linear — §6
3. Quadratic — §7.1
4. Polynomial (higher degree) — §7.2–7.5
5. Factored form — §7.5
6. Rational — §8
7. Radical — §9
8. Exponential — §10
9. Logarithmic — §11
10. Trigonometric — §12
11. Mixed / unclassified — attempt transformations, then return unevaluated

### 17.2 Recursion and composition

**PIRF-Solve-E-010** [Event-driven]
When a solving rule produces a new `Solve` call in its result (e.g. after
a substitution), the engine shall recursively solve the inner equation.

**PIRF-Solve-E-011** [Ubiquitous]
The engine shall enforce a configurable maximum recursion depth
(default: 20) for solving.

**PIRF-Solve-E-012** [Unwanted behaviour]
If recursion exceeds the limit, the engine shall return the equation
in unevaluated form.

### 17.3 Simplification of solutions

**PIRF-Solve-E-020** [Event-driven]
When the engine produces a solution, it shall apply PIRF-S simplification
to each solution expression.

**PIRF-Solve-E-021** [Event-driven]
When the engine produces duplicate solutions (after simplification),
it shall remove duplicates from the solution set.

---

## 18. Requirements — Solving file format and taxonomy

### 18.1 Taxonomy

**PIRF-Solve-T-001** [Ubiquitous]
Equation solving rules shall be organised as Section 12 of the unified
PIRF hierarchy:

| Section | Title | Rule count (est.) |
|---------|-------|-------------------|
| 12.1 | Equation normalisation | ~5 |
| 12.2 | Linear equations | ~5 |
| 12.3 | Polynomial equations | ~20 |
| 12.4 | Rational equations | ~5 |
| 12.5 | Radical equations | ~8 |
| 12.6 | Exponential equations | ~8 |
| 12.7 | Logarithmic equations | ~5 |
| 12.8 | Trigonometric equations | ~15 |
| 12.9 | Systems of equations | ~10 |
| 12.10 | Inequalities | ~15 |
| 12.11 | Ordinary differential equations | ~25 |
| 12.12 | Extraneous solution checking | ~5 |
| **Total** | | **~126** |

### 18.2 File organisation

**PIRF-Solve-F-001** [Ubiquitous]
Equation solving rule files shall be placed in a `solve/` directory:

```
PIRF/
├── rules/              # Integration (§1–9)
├── deriv/              # Differentiation (§10)
├── simplify/           # Simplification (§11)
├── solve/              # Equation solving (§12)
│   ├── meta.json
│   ├── 12.1-normalisation.json
│   ├── 12.2-linear.json
│   ├── 12.3-polynomial/
│   │   ├── 12.3.1-quadratic.json
│   │   ├── 12.3.2-cubic.json
│   │   ├── 12.3.3-quartic.json
│   │   ├── 12.3.4-higher-degree.json
│   │   └── 12.3.5-factored-forms.json
│   ├── 12.4-rational.json
│   ├── 12.5-radical.json
│   ├── 12.6-exponential.json
│   ├── 12.7-logarithmic.json
│   ├── 12.8-trigonometric/
│   │   ├── 12.8.1-basic-forms.json
│   │   ├── 12.8.2-quadratic-trig.json
│   │   └── 12.8.3-linear-combinations.json
│   ├── 12.9-systems/
│   │   ├── 12.9.1-linear-systems.json
│   │   └── 12.9.2-nonlinear-systems.json
│   ├── 12.10-inequalities/
│   │   ├── 12.10.1-linear.json
│   │   ├── 12.10.2-polynomial.json
│   │   ├── 12.10.3-rational.json
│   │   └── 12.10.4-absolute-value.json
│   ├── 12.11-ode/
│   │   ├── 12.11.1-separable.json
│   │   ├── 12.11.2-linear-first-order.json
│   │   ├── 12.11.3-exact.json
│   │   ├── 12.11.4-bernoulli.json
│   │   ├── 12.11.5-second-order-constant.json
│   │   ├── 12.11.6-variation-of-parameters.json
│   │   └── 12.11.7-initial-boundary.json
│   └── 12.12-extraneous-check.json
└── tests/
```

**PIRF-Solve-F-002** [Ubiquitous]
Equation solving rule IDs shall use the range 120001–129999.

**PIRF-Solve-F-003** [Ubiquitous]
ODE solving rule IDs shall use the sub-range 120200–120299.

### 18.3 Load priority

**PIRF-Solve-F-010** [Ubiquitous]
The loading order shall be:

1. **12.1** — Normalisation (applied first to all equations)
2. **12.2** — Linear (simplest, most common)
3. **12.3** — Polynomial (by increasing degree)
4. **12.4** — Rational
5. **12.5** — Radical
6. **12.6** — Exponential
7. **12.7** — Logarithmic
8. **12.8** — Trigonometric
9. **12.9** — Systems
10. **12.10** — Inequalities
11. **12.11** — ODE (depends on integration/differentiation)
12. **12.12** — Extraneous check (applied last, post-processing)

---

## 19. Requirements — Solving test suite

**PIRF-Solve-V-001** [Ubiquitous]
The format shall include a solving test suite in `tests-solve/`.

**PIRF-Solve-V-002** [Ubiquitous]
Each test shall contain: `id`, `equation` (PIRF-Expr), `variable`
(string or list for systems), `expected_solutions` (PIRF-Expr),
`domain` (optional: `"Reals"` or `"Complexes"`).

```json
{
  "id": 420001,
  "equation": ["Equal", ["Add", ["Power", "x", 2], ["Negate", 1]], 0],
  "variable": "x",
  "expected_solutions": ["Solution", 1, -1],
  "domain": "Reals"
}
```

**PIRF-Solve-V-003** [Ubiquitous]
ODE tests shall additionally contain `initial_conditions` (optional list).

```json
{
  "id": 420201,
  "equation": ["Equal", ["D", "y", "x"], ["Multiply", 2, "x"]],
  "variable": "y",
  "independent_variable": "x",
  "expected_solutions": ["Equal", "y", ["Add", ["Power", "x", 2], ["C", 1]]],
  "initial_conditions": [["Equal", ["y", 0], 1]]
}
```

**PIRF-Solve-V-004** [Event-driven]
When a test is executed, solutions shall be verified by **substitution**
into the original equation (symbolic equivalence).

**PIRF-Solve-V-005** [Ubiquitous]
The solving test suite shall cover a minimum of **400 test problems**:
at least 50 linear, 80 polynomial, 30 rational, 30 radical, 30
exponential, 20 logarithmic, 40 trigonometric, 30 systems, 40
inequalities, and 50 ODEs.

**PIRF-Solve-V-006** [Unwanted behaviour]
If the engine produces no result within configurable timeout
(default: 30s), the test shall be marked as failed.

---

## 20. Requirements — Portability

**PIRF-Solve-P-001** [Ubiquitous]
All solving rules and test files shall be independent of any
programming language.

**PIRF-Solve-P-002** [Ubiquitous]
The Phrasebook architecture (PIRF §13.3) shall apply: the Phrasebook
translates `Solution[...]` and `ParametricSolution[...]` to the host
CAS's native solution representation.

**PIRF-Solve-P-003** [Ubiquitous]
The Phrasebook's required algorithmic kernel for solving is:

| Algorithm | Purpose | Used by |
|-----------|---------|---------|
| Gaussian elimination | Linear systems | §13 |
| Polynomial GCD / resultant | Variable elimination | §13.2 |
| Numeric root-finding | Degree ≥ 5, `NSolve` | §7.4 |
| Numeric substitution | Extraneous check fallback | §16 |

---

## 21. Traceability matrix

| Category | Identifiers | Count |
|----------|------------|-------|
| Operators (Solve-X) | PIRF-Solve-X-001 to X-010 | 3 (with tables) |
| Normalisation (Solve-R-001–004) | PIRF-Solve-R-001 to R-004 | 4 |
| Linear (Solve-R-010–012) | PIRF-Solve-R-010 to R-012 | 3 |
| Polynomial (Solve-R-020–056) | PIRF-Solve-R-020 to R-056 | 11 |
| Rational (Solve-R-060–062) | PIRF-Solve-R-060 to R-062 | 3 |
| Radical (Solve-R-070–073) | PIRF-Solve-R-070 to R-073 | 4 |
| Exponential (Solve-R-080–083) | PIRF-Solve-R-080 to R-083 | 4 |
| Logarithmic (Solve-R-090–092) | PIRF-Solve-R-090 to R-092 | 3 |
| Trigonometric (Solve-R-100–121) | PIRF-Solve-R-100 to R-121 | 7 |
| Systems (Solve-R-130–141) | PIRF-Solve-R-130 to R-141 | 6 |
| Inequalities (Solve-R-150–170) | PIRF-Solve-R-150 to R-170 | 6 |
| ODE (Solve-R-200–272) | PIRF-Solve-R-200 to R-272 | 14 |
| Extraneous (Solve-R-300–303) | PIRF-Solve-R-300 to R-303 | 4 |
| Engine (Solve-E) | PIRF-Solve-E-001 to E-021 | 8 |
| Taxonomy (Solve-T) | PIRF-Solve-T-001 | 1 |
| File format (Solve-F) | PIRF-Solve-F-001 to F-010 | 4 |
| Test suite (Solve-V) | PIRF-Solve-V-001 to V-006 | 6 |
| Portability (Solve-P) | PIRF-Solve-P-001 to P-003 | 3 |
| **Total** | | **94** |

---

## 22. EARS pattern distribution

| EARS pattern | Count | % |
|-------------|-------|---|
| Ubiquitous | 40 | 43% |
| Event-driven (When) | 42 | 45% |
| Unwanted behaviour (If…then) | 8 | 8% |
| Optional feature (Where) | 4 | 4% |
| Complex (While…When) | 0 | 0% |
| State-driven (While) | 0 | 0% |

> **Design note — High event-driven ratio.** Solving is inherently
> reactive (classify → dispatch → transform → recurse), which explains
> the higher proportion of event-driven requirements compared to the
> simplification spec.

---

## Annex F — Solving operator mapping tables (normative)

### F.1 Equation operators

| PIRF-Expr | Mathematica | SymPy | Julia/Symbolics |
|-----------|-------------|-------|-----------------|
| `Equal[a, b]` | `a == b` | `Eq(a, b)` | `a ~ b` |
| `Less[a, b]` | `a < b` | `a < b` | `a < b` |
| `Solve[eq, x]` | `Solve[eq, x]` | `solve(eq, x)` | `Symbolics.solve_for(eq, x)` |
| `DSolve[ode, y, x]` | `DSolve[ode, y, x]` | `dsolve(ode, y(x))` | `ModelingToolkit` |
| `Solution[s1, s2]` | `{{x->s1}, {x->s2}}` | `FiniteSet(s1, s2)` | `[s1, s2]` |
| `NoSolution` | `{}` | `EmptySet` | `[]` |
| `C[n]` | `C[n]` | `C1, C2, ...` | `C₁, C₂, ...` |

### F.2 Domain symbols

| PIRF-Expr | Mathematica | SymPy | OpenMath CD |
|-----------|-------------|-------|-------------|
| `Integers` | `Integers` | `S.Integers` | `setname1#Z` |
| `Rationals` | `Rationals` | `S.Rationals` | `setname1#Q` |
| `Reals` | `Reals` | `S.Reals` | `setname1#R` |
| `Complexes` | `Complexes` | `S.Complexes` | `setname1#C` |

---

## Annex G — Updated dependency diagram

```
┌─────────────────────────────────────────────────────────────────────┐
│                     PIRF-Expr (shared language)                      │
├─────────────────────────────────────────────────────────────────────┤
│  PIRF-Solve  │  PIRF-S         │  PIRF-D        │  PIRF           │
│  (§12)       │  (§11)          │  (§10)         │  (§1–9)         │
│  Solving     │  Simplification │  Derivation    │  Integration    │
│  ~126 rules  │  ~225 rules     │  ~100 rules    │  ~7,800 rules   │
├──────────────┴─────────────────┴────────────────┴─────────────────┤
│  Phrasebook kernel: GCD, factorisation, Gaussian elim., numeric   │
│  root-finding, numeric eval, pattern matching                      │
├─────────────────────────────────────────────────────────────────────┤
│  Host CAS: Julia / Python / Rust / JS / Mathematica / ...          │
└─────────────────────────────────────────────────────────────────────┘

Dependencies:
  PIRF-Solve ──depends on──▶ PIRF-S (simplification of solutions)
  PIRF-Solve ──depends on──▶ PIRF-D (ODE solving needs derivatives)
  PIRF-Solve ──depends on──▶ PIRF  (ODE solving needs integration)
  PIRF       ──depends on──▶ PIRF-Solve (partial fractions need roots)
```
