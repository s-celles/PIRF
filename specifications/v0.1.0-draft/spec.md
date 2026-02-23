# EARS Specification — Portable Format for Symbolic Integration Rules

**Project:** RUBI Portable Integration Rules Format (RUBI-PIRF)
**Version:** 0.1.0-draft
**Date:** 2026-02-23
**Method:** EARS (Easy Approach to Requirements Syntax) — Alistair Mavin, 2009
**System:** Any CAS implementing RUBI-style symbolic integration
**Inspired by:** RUBI 4.16+ / RUBI 5 (Albert Rich)

---

## 1. Purpose and scope

This document specifies the requirements for a **portable JSON format**
capable of representing RUBI-style symbolic integration rules, independently of
any programming language or computer algebra system.

The format is designed to be consumable by any CAS: Julia, Python/SymPy,
Java/SymJa, Rust, JavaScript, Mathematica, and others.

Today, RUBI rules exist only in Mathematica-native formats (.nb, .m). Every port
(SymJa, SymPy) reparses those files with ad-hoc converters. This specification
aims to be the **first language-neutral interchange format** for integration rule
knowledge, covering the full set of ~7,800 rules and 72,000+ test problems.

> **Note on rule count.** RUBI 4.16 contained approximately 6,700 rules.
> The ongoing RUBI 5 redesign has expanded this to approximately 7,800 rules.
> This specification targets RUBI 5's rule set. All references to "the full
> rule set" mean approximately 7,800 rules unless otherwise noted.

### 1.1 Standardisation context

Mathematical expression formats in the current landscape:

| Format | Status | Syntax | Strengths | Weaknesses for this spec |
|--------|--------|--------|-----------|--------------------------|
| **MathML Content** | W3C Rec. / ISO | XML | Official standard, HTML5 | Extremely verbose, no pattern matching |
| **OpenMath 2.0** | Standard (OM Society) | XML, binary, JSON | Formal semantics via Content Dictionaries | Complex, not designed for rewriting rules |
| **MathJSON** (MathLive) | De facto, MIT | JSON | Lightweight, readable, S-expr in JSON | Not standardised, single maintainer |
| **S-expressions** | None | Text | Universal (Lisp heritage), minimal | No shared naming conventions |

None of these formats addresses **integration rule representation** (pattern
matching with wildcards, constraint predicates, recursive rule application).

### 1.2 Standardisation strategy

This specification adopts a **hybrid approach** ("Profile + Own the Spec"):

1. **Define PIRF-Expr** formally (JSON Schema, ABNF grammar) as a self-contained
   micro-standard within this specification.
2. **Align operator names** with MathLive MathJSON conventions (PascalCase, same
   names: `Add`, `Sin`, `Power`, etc.) for de facto compatibility.
3. **Provide mapping tables** to OpenMath Content Dictionaries and Content MathML
   for formal interoperability (see Annex A).
4. **Extend with RUBI-specific operators** (`Int`, `Dist`, `Subst`, `Simp`,
   wildcards, utility functions, special functions) that exist in no current
   standard — because no existing format covers pattern matching for symbolic
   integration.

This specification does not *depend* on MathJSON (which may evolve or be
abandoned), but is *compatible* with it and *traceable* to formal standards.

### 1.3 Key differences with MathLive MathJSON

While PIRF-Expr is aligned with MathLive MathJSON on core operator names,
the following architectural differences exist and are intentional:

| Aspect | MathLive MathJSON | PIRF-Expr | Rationale |
|--------|-------------------|-----------|-----------|
| **Expression syntax** | Dual: `["Add", 1, "x"]` (compact) or `{"fn": ["Add", 1, "x"]}` (with metadata) | Compact form only: `["Add", 1, "x"]` | Simplicity; metadata not needed in rule files |
| **Inverse trig** | Compositional: `["Apply", ["InverseFunction", "Sin"], "x"]` | Direct operator: `["Asin", "x"]` | RUBI rules reference `ArcSin` as atomic function; compositional form would complicate pattern matching |
| **Numbers** | Extended: `{"num": "3.14..."}` for arbitrary precision, `"NaN"`, `"+Infinity"`, `"-Infinity"`, repeating decimals `"1.(3)"` | JSON native numbers only | Integration rules use symbolic constants (`Pi`, `E`), not high-precision literals |
| **Symbols** | Unicode-aware with NFC normalisation, emoji support, backtick-wrapped symbols | ASCII identifiers with wildcard suffixes (`x_`, `a.`) | Pattern matching requires simple, predictable symbol grammar |
| **Strings** | Apostrophe-delimited `"'text'"` or `{"str": "text"}` | Not used | Rule files contain no prose strings |

These differences are documented so that Phrasebook implementors know exactly
where translation is required (see Annex A).

---

## 2. Glossary

| Term | Definition |
|------|-----------|
| **PIRF** | Portable Integration Rules Format — the format specified herein |
| **PIRF-Expr** | The expression sub-language defined by this specification for mathematical expressions |
| **Rule** | Triple (pattern, constraints, result) describing an integral transformation |
| **Pattern** | PIRF-Expr with wildcards describing the integrand form |
| **Constraint** | PIRF-Expr predicate over bound variables (FreeQ, IntegerQ, etc.) |
| **Result** | PIRF-Expr of the antiderivative or recursive call to the integrator |
| **Wildcard** | Pattern matching symbol: `x_` (mandatory), `a.` (optional with default), `m_integer` (typed) |
| **Loader** | Software module that reads JSON files and builds the decision tree |
| **Engine** | Software module that applies rules to a symbolic expression |
| **Taxonomy** | Hierarchy of rule categories matching RUBI's 9-section structure |
| **Test suite** | Set of (integrand, expected antiderivative) pairs for validation |
| **Content Dictionary (CD)** | OpenMath mechanism assigning semantics to symbols |
| **Phrasebook** | Adapter translating between PIRF-Expr and a host CAS internal representation |
| **Utility function** | Algorithmic function (e.g. NormalizeIntegrand) that each Phrasebook must implement |
| **Inert function** | A trig/hyp function temporarily deactivated during pattern matching to prevent premature CAS evaluation |
| **Load manifest** | Ordered list of file paths in `meta.json` defining the exact sequence in which rule files must be loaded |

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

## 4. Requirements — PIRF-Expr expression language

### 4.1 Formal grammar

**PIRF-X-001** [Ubiquitous]
The PIRF-Expr language shall be defined by the following ABNF grammar (RFC 5234):

```abnf
pirf-expr    = number / symbol / string / function-expr

number       = json-number              ; as defined in RFC 8259 §6

symbol       = json-string              ; matching the symbol grammar below

string       = json-string              ; any JSON string not matching symbol grammar

function-expr = "[" operator "," pirf-expr *("," pirf-expr) "]"

operator     = json-string              ; matching /^[A-Z][a-zA-Z0-9$]*$/

; Symbol sub-grammar:
plain-symbol    = ALPHA *(ALPHA / DIGIT / "_" / "$")  ; e.g. "x", "alpha", "$base$"
wildcard-req    = plain-symbol "_" [type-tag]          ; e.g. "m_", "n_integer"
wildcard-opt    = plain-symbol "." [default-spec]      ; e.g. "a.", "b.", "m.3"
default-spec    = 1*DIGIT / "-" 1*DIGIT / "1"         ; explicit default value
type-tag        = "integer" / "rational" / "positive" / "negative" / "complex"
```

> **Design note:** The wildcard-opt grammar includes an optional
> `default-spec` to support arbitrary default values (not just 0 or 1).
> The `$` character is permitted in symbol names to accommodate RUBI's
> internal fluid variables (e.g. `$base$`, `$expon$`).

**PIRF-X-002** [Ubiquitous]
The PIRF-Expr language shall be a strict subset of JSON — every valid PIRF-Expr shall be a valid JSON value.

**PIRF-X-003** [Ubiquitous]
The PIRF-Expr language shall be self-contained: its grammar, operator catalogue, and semantics shall be fully defined within this specification, without normative reference to any external format.

**PIRF-X-004** [Ubiquitous]
The PIRF-Expr operator names shall follow PascalCase convention, starting with an uppercase letter A-Z.

### 4.2 Operator catalogue — Core arithmetic

**PIRF-X-010** [Ubiquitous]
PIRF-Expr shall define the following **core arithmetic operators**:

| Operator | Arity | Semantics | OpenMath CD | MathML Content |
|----------|-------|-----------|-------------|----------------|
| `Add` | 2+ | Sum | `arith1#plus` | `<plus/>` |
| `Subtract` | 2 | Difference | `arith1#minus` | `<minus/>` |
| `Multiply` | 2+ | Product | `arith1#times` | `<times/>` |
| `Divide` | 2 | Quotient | `arith1#divide` | `<divide/>` |
| `Power` | 2 | Exponentiation | `arith1#power` | `<power/>` |
| `Negate` | 1 | Unary minus | `arith1#unary_minus` | `<minus/>` (unary) |
| `Sqrt` | 1 | Square root | `arith1#root` (n=2) | `<root/>` |
| `Abs` | 1 | Absolute value | `arith1#abs` | `<abs/>` |
| `Factorial` | 1 | Factorial | `integer1#factorial` | `<factorial/>` |

### 4.3 Operator catalogue — Trigonometric functions

**PIRF-X-011** [Ubiquitous]
PIRF-Expr shall define the following **trigonometric function operators** (the full set of 6 circular functions):

| Operator | Arity | OpenMath CD | MathML Content |
|----------|-------|-------------|----------------|
| `Sin` | 1 | `transc1#sin` | `<sin/>` |
| `Cos` | 1 | `transc1#cos` | `<cos/>` |
| `Tan` | 1 | `transc1#tan` | `<tan/>` |
| `Cot` | 1 | `transc1#cot` | `<cot/>` |
| `Sec` | 1 | `transc1#sec` | `<sec/>` |
| `Csc` | 1 | `transc1#csc` | `<csc/>` |

### 4.4 Operator catalogue — Hyperbolic functions

**PIRF-X-012** [Ubiquitous]
PIRF-Expr shall define the following **hyperbolic function operators** (the full set of 6 hyperbolic functions):

| Operator | Arity | OpenMath CD | MathML Content |
|----------|-------|-------------|----------------|
| `Sinh` | 1 | `transc1#sinh` | `<sinh/>` |
| `Cosh` | 1 | `transc1#cosh` | `<cosh/>` |
| `Tanh` | 1 | `transc1#tanh` | `<tanh/>` |
| `Coth` | 1 | `transc1#coth` | `<coth/>` |
| `Sech` | 1 | `transc1#sech` | `<sech/>` |
| `Csch` | 1 | `transc1#csch` | `<csch/>` |

### 4.5 Operator catalogue — Inverse trigonometric functions

**PIRF-X-013** [Ubiquitous]
PIRF-Expr shall define the following **inverse trigonometric function operators** (the full set of 6 inverse circular functions):

| Operator | Arity | OpenMath CD | MathML Content |
|----------|-------|-------------|----------------|
| `Asin` | 1 | `transc1#arcsin` | `<arcsin/>` |
| `Acos` | 1 | `transc1#arccos` | `<arccos/>` |
| `Atan` | 1-2 | `transc1#arctan` | `<arctan/>` |
| `Acot` | 1 | `transc1#arccot` | `<arccot/>` |
| `Asec` | 1 | `transc1#arcsec` | `<arcsec/>` |
| `Acsc` | 1 | `transc1#arccsc` | `<arccsc/>` |

> **Design note — Inverse trigonometric operators.**
> MathLive MathJSON does not define `Arcsin`/`Arccos`/`Arctan` as direct
> operators. Instead, it uses a compositional form:
> `["Apply", ["InverseFunction", "Sin"], "x"]`. This specification deliberately
> defines `Asin`, `Acos`, `Atan`, etc. as **first-class atomic operators** because:
> (a) RUBI rules reference `ArcSin[x]` as an atomic function head;
> (b) pattern matching on `["Asin", "x_"]` is simpler than on a nested
> `["Apply", ["InverseFunction", ...], ...]` structure;
> (c) the Mathematica-to-PIRF converter maps `ArcSin` → `Asin` directly.
> The MathJSON Phrasebook shall translate `["Asin", x]` to
> `["Apply", ["InverseFunction", "Sin"], x]` and vice versa.

### 4.6 Operator catalogue — Inverse hyperbolic functions

**PIRF-X-014** [Ubiquitous]
PIRF-Expr shall define the following **inverse hyperbolic function operators** (the full set of 6 inverse hyperbolic functions):

| Operator | Arity | OpenMath CD | MathML Content |
|----------|-------|-------------|----------------|
| `Asinh` | 1 | `transc1#arcsinh` | `<arcsinh/>` |
| `Acosh` | 1 | `transc1#arccosh` | `<arccosh/>` |
| `Atanh` | 1 | `transc1#arctanh` | `<arctanh/>` |
| `Acoth` | 1 | `transc1#arccoth` | `<arccoth/>` |
| `Asech` | 1 | `transc1#arcsech` | `<arcsech/>` |
| `Acsch` | 1 | `transc1#arccsch` | `<arccsch/>` |

### 4.7 Operator catalogue — Exponential and logarithmic functions

**PIRF-X-015** [Ubiquitous]
PIRF-Expr shall define the following **exponential and logarithmic operators**:

| Operator | Arity | Semantics | OpenMath CD | MathML Content |
|----------|-------|-----------|-------------|----------------|
| `Exp` | 1 | Exponential function | `transc1#exp` | `<exp/>` |
| `Log` | 1-2 | Natural log (1-arg) or log base b (2-arg) | `transc1#ln` / `transc1#log` | `<ln/>` / `<log/>` |

### 4.8 Operator catalogue — Special functions

**PIRF-X-016** [Ubiquitous]
PIRF-Expr shall define the following **special function operators** required by RUBI sections 8.1–8.10:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `PolyLog` | 2 | Polylogarithm Li_n(z) |
| `ProductLog` | 1-2 | Lambert W function |
| `Erf` | 1 | Error function |
| `Erfi` | 1 | Imaginary error function |
| `Erfc` | 1 | Complementary error function |
| `Gamma` | 1-2 | Gamma function / incomplete gamma |
| `LogGamma` | 1 | Log-gamma function |
| `Beta` | 2-3 | Beta function / incomplete beta |
| `Hypergeometric2F1` | 4 | Gauss hypergeometric ₂F₁(a,b;c;z) |
| `HypergeometricPFQ` | 3 | Generalized hypergeometric pFq |
| `EllipticF` | 2 | Elliptic integral of the first kind |
| `EllipticE` | 1-2 | Elliptic integral of the second kind |
| `EllipticPi` | 3 | Elliptic integral of the third kind |
| `EllipticK` | 1 | Complete elliptic integral of the first kind |
| `ExpIntegralE` | 2 | Generalized exponential integral E_n(z) |
| `ExpIntegralEi` | 1 | Exponential integral Ei(z) |
| `LogIntegral` | 1 | Logarithmic integral li(z) |
| `SinIntegral` | 1 | Sine integral Si(z) |
| `CosIntegral` | 1 | Cosine integral Ci(z) |
| `SinhIntegral` | 1 | Hyperbolic sine integral Shi(z) |
| `CoshIntegral` | 1 | Hyperbolic cosine integral Chi(z) |
| `FresnelS` | 1 | Fresnel integral S(z) |
| `FresnelC` | 1 | Fresnel integral C(z) |
| `Zeta` | 1-2 | Riemann zeta / Hurwitz zeta |
| `AppellF1` | 6 | Appell hypergeometric F₁ |
| `BesselJ` | 2 | Bessel function of the first kind J_n(z) |
| `BesselY` | 2 | Bessel function of the second kind Y_n(z) |
| `BesselI` | 2 | Modified Bessel function of the first kind I_n(z) |
| `BesselK` | 2 | Modified Bessel function of the second kind K_n(z) |

> **Design note:** Bessel functions (`BesselJ`, `BesselY`, `BesselI`,
> `BesselK`) are included to cover RUBI section 8.10. Although this section
> is currently commented out in `Rubi.m`, the rules exist in the notebooks
> and are included in the IntegrationRules PDF repository.

### 4.9 Operator catalogue — Mathematical constants

**PIRF-X-017** [Ubiquitous]
PIRF-Expr shall define the following **mathematical constant symbols**:

| Symbol | Semantics | OpenMath CD |
|--------|-----------|-------------|
| `Pi` | π ≈ 3.14159… | `nums1#pi` |
| `E` | Euler's number e ≈ 2.71828… | `nums1#e` |
| `I` | Imaginary unit √(−1) | `nums1#i` |
| `Infinity` | Positive infinity ∞ | `nums1#infinity` |
| `ComplexInfinity` | Complex infinity (direction undefined) | *(none)* |
| `GoldenRatio` | Golden ratio φ ≈ 1.61803… | *(none)* |
| `EulerGamma` | Euler–Mascheroni constant γ ≈ 0.57721… | `nums1#gamma` |

### 4.10 Operator catalogue — Integration-specific operators

**PIRF-X-018** [Ubiquitous]
PIRF-Expr shall define the following **integration-specific operators** with no equivalents in existing standards:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `Int` | 2 | Recursive integration call: ∫ $1 d$2 |
| `Dist` | 3 | Constant distribution: $1 · ∫ $2 d$3 |
| `Subst` | 3 | Substitution: $1 with $2 → $3 |
| `Simp` | 2 | Simplification of $1 w.r.t. $2 |
| `ExpandIntegrand` | 2 | Expand $1 before integration w.r.t. $2 |
| `Unintegrable` | 2 | Expression is not integrable in closed form |
| `CannotIntegrate` | 2 | Engine is unable to integrate expression |

### 4.11 Operator catalogue — Utility functions

**PIRF-X-019** [Ubiquitous]
PIRF-Expr shall define the following **utility function operators** used in
RUBI rule results and constraints. These are algorithmic functions that each
Phrasebook must implement; the format defines their calling convention.

#### 4.11.1 Normalization and simplification

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `NormalizeIntegrand` | 2 | Normalize u to a standard form recognizable by rules, w.r.t. x |
| `SimplifyIntegrand` | 2 | Simplify u and return in standard form, w.r.t. x |
| `SimplifyAntiderivative` | 2 | Simplify antiderivative u, ensuring continuity, w.r.t. x |
| `NormalizeLeadTermSigns` | 1 | Ensure lead terms of sum factors have positive coefficients |
| `NormalizeSumFactors` | 1 | Normalize numeric coefficients of lead terms in sum factors |
| `AbsorbMinusSign` | 1 | Distribute minus sign into a sum factor raised to odd power |
| `FixSimplify` | 1 | Post-process simplification fixing known CAS issues |
| `SmartSimplify` | 1 | Simplification avoiding problematic transformations |
| `TogetherSimplify` | 1 | Combine over common denominator then simplify |
| `ContentFactor` | 1 | Factor out greatest common numeric content |

#### 4.11.2 Polynomial and algebraic manipulation

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `Coefficient` | 2-3 | Polynomial coefficient: Coefficient(expr, x) or Coefficient(expr, x, n) |
| `Exponent` | 2 | Polynomial degree of expr in x |
| `IntPart` | 1 | Integer part of a number |
| `FracPart` | 1 | Fractional part of a number |
| `Together` | 1 | Combine over common denominator |
| `Apart` | 1-2 | Partial fraction decomposition |
| `Cancel` | 1 | Cancel common factors in numerator/denominator |
| `Factor` | 1 | Factor expression |
| `Expand` | 1 | Expand expression |
| `Numerator` | 1 | Numerator of expression |
| `Denominator` | 1 | Denominator of expression |
| `SmartNumerator` | 1 | Numerator that handles negative powers intelligently |
| `SmartDenominator` | 1 | Denominator that handles negative powers intelligently |
| `Rt` | 2 | RUBI's n-th root choosing the simplest real/complex form |

#### 4.11.3 Trigonometric manipulation

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `TrigReduce` | 1 | Reduce products of trig functions to sums |
| `TrigExpand` | 1 | Expand trig of sums to products |
| `TrigToExp` | 1 | Convert trig to exponential form |
| `ExpToTrig` | 1 | Convert exponential to trig form |
| `SmartTrigExpand` | 1 | Trig expansion avoiding problematic forms |
| `SmartTrigReduce` | 1 | Trig reduction avoiding problematic forms |
| `TrigSimplifyAux` | 1 | Auxiliary trig simplification |
| `ActivateTrig` | 1 | Convert inert trig function forms to active (evaluable) forms |
| `DeactivateTrig` | 1 | Convert active trig function forms to inert (non-evaluable) forms for pattern matching |

> **Design note — Inert/active trig functions.**
> RUBI temporarily deactivates trig functions during pattern matching to
> prevent premature CAS evaluation. Sections 4.7.5 ("Inert trig functions")
> and 4.7.9 ("Active trig functions") in the RUBI rule set contain the
> normalization rules that perform this conversion. `ActivateTrig` and
> `DeactivateTrig` are the Phrasebook-implemented functions that handle
> this mechanism. Inert forms use lowercased heads (e.g. `sin` instead of
> `Sin`) or a wrapper operator; each Phrasebook defines the mapping.

#### 4.11.4 Structural decomposition

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `LeadTerm` | 1 | First term of a sum |
| `RemainingTerms` | 1 | All terms except the first |
| `LeadFactor` | 1 | First factor of a product |
| `RemainingFactors` | 1 | All factors except the first |
| `LeadBase` | 1 | Base of the leading power factor |
| `LeadDegree` | 1 | Exponent of the leading power factor |
| `MergeableFactorQ` | 3 | Can two power factors be merged? |
| `MergeFactor` | 3 | Merge two power factors |
| `MergeFactors` | 2 | Merge compatible factors in a product |

#### 4.11.5 Substitution helpers

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `SubstFor` | 3 | Find substitution to express u as function of v w.r.t. x |
| `SubstForFractionalPowerOfLinear` | 2 | Substitution for (a+bx)^(1/n) |
| `SubstForFractionalPowerOfQuotientOfLinears` | 2 | Substitution for ((a+bx)/(c+dx))^(1/n) |
| `SubstForInverseFunction` | 2 | Substitution for inverse trig/hyp function |
| `SubstForExpn` | 2 | Substitution for exponential expression |

#### 4.11.6 Calculus and general

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `D` | 2 | Symbolic derivative of $1 w.r.t. $2 (Mathematica built-in) |
| `Dif` | 2 | RUBI's own symbolic derivative (distinct from D; avoids built-in side effects) |
| `Map` | 2 | Apply function to sub-expressions |
| `Simplify` | 1 | General simplification |
| `FullSimplify` | 1 | Aggressive simplification |
| `ReplaceAll` | 2 | Substitution: $1 /. $2 |
| `Mods` | 2 | Modular arithmetic function used in RUBI rules |

### 4.12 Operator catalogue — Structural/utility operators

**PIRF-X-020** [Ubiquitous]
PIRF-Expr shall define the following **structural/utility operators**:

| Operator | Arity | Semantics |
|----------|-------|-----------|
| `List` | 0+ | Ordered list of expressions |
| `Piecewise` | 2+ | Piecewise-defined expression |
| `Condition` | 2 | Expression with side condition |
| `If` | 2-3 | Conditional: If(test, true-branch [, false-branch]) |
| `With` | 2 | Local variable binding (for rule results) |
| `Module` | 2 | Local scoped variable binding (broader than With) |
| `CompoundExpression` | 2+ | Sequential evaluation |
| `Rule` | 2 | Rewrite rule: lhs → rhs |
| `Set` | 2 | Assignment |
| `Floor` | 1 | Floor function |
| `Ceiling` | 1 | Ceiling function |
| `Round` | 1 | Round to nearest integer |
| `Mod` | 2 | Modulo |
| `Max` | 2+ | Maximum |
| `Min` | 2+ | Minimum |
| `Sign` | 1 | Sign of expression (−1, 0, or 1) |
| `Conjugate` | 1 | Complex conjugate |
| `Re` | 1 | Real part |
| `Im` | 1 | Imaginary part |
| `Head` | 1 | Function head of an expression |
| `Length` | 1 | Number of arguments |
| `Part` | 2+ | Extract sub-expression by index |
| `Apply` | 2 | Apply function head to a list of arguments |
| `Scan` | 2 | Apply function to each element for side effects |
| `Catch` | 1 | Catch a thrown value |
| `Throw` | 1 | Throw a value to enclosing Catch |
| `Hold` | 1 | Prevent evaluation of argument |
| `HoldForm` | 1 | Prevent evaluation and display unevaluated |
### 4.13 Extensibility

**PIRF-X-025** [Optional feature]
Where a host system defines additional operators beyond the PIRF-Expr catalogue, the host system shall declare them in the `"extensions"` field of `meta.json` with their name, arity, and informal semantics.

**PIRF-X-026** [Unwanted behaviour]
If a PIRF-Expr expression contains an operator not present in the core catalogue and not declared in `meta.json` extensions, then the Loader shall emit a warning and treat the operator as an opaque function symbol.

### 4.14 Compatibility and mapping

**PIRF-X-030** [Ubiquitous]
The specification shall include a normative **Annex A** providing bidirectional mapping tables between PIRF-Expr operators and:
  (a) MathLive MathJSON Standard Library operator names,
  (b) OpenMath Content Dictionary symbols,
  (c) Content MathML elements.

**PIRF-X-031** [Ubiquitous]
For core arithmetic and direct trigonometric operators (excluding inverse trigonometric functions), PIRF-Expr operator names shall be identical to MathLive MathJSON operator names to maximise de facto compatibility.

**PIRF-X-031a** [Ubiquitous]
For inverse trigonometric and inverse hyperbolic functions, PIRF-Expr shall use shortened names (`Asin`, `Acos`, `Atan`, `Asinh`, etc.) rather than the MathJSON compositional form. The Annex A mapping table shall document this divergence explicitly.

**PIRF-X-032** [Ubiquitous]
This specification shall clearly state that PIRF-Expr is a self-contained format: compliance does not require any MathJSON, OpenMath, or MathML implementation.

**PIRF-X-033** [Optional feature]
Where a Phrasebook (adapter) to OpenMath is provided, the Phrasebook shall map PIRF-Expr operators to the OpenMath CD symbols listed in the Annex A mapping table.

**PIRF-X-034** [Optional feature]
Where a Phrasebook to Content MathML is provided, the Phrasebook shall map PIRF-Expr operators to the Content MathML elements listed in the Annex A mapping table.

### 4.15 Versioning and stability

**PIRF-X-040** [Ubiquitous]
The PIRF-Expr operator catalogue shall be versioned independently using SemVer, within the overall schema version.

**PIRF-X-041** [Ubiquitous]
Removal of an operator from the core catalogue shall constitute a breaking change and shall increment the major version.

**PIRF-X-042** [Ubiquitous]
Addition of new operators to the core catalogue shall constitute a minor change and shall increment the minor version.

**PIRF-X-043** [Ubiquitous]
In case of divergence between a future MathLive MathJSON version and PIRF-Expr, the PIRF-Expr definition shall take precedence for compliant implementations.

---

## 5. Requirements — JSON file format

### 5.1 General structure

**PIRF-F-001** [Ubiquitous]
The format shall use JSON (RFC 8259) as its serialisation syntax.

**PIRF-F-002** [Ubiquitous]
The format shall use UTF-8 encoding without BOM.

**PIRF-F-003** [Ubiquitous]
Each rule file shall contain a root JSON object with the mandatory fields `$schema`, `section`, `title`, and `rules`.

**PIRF-F-004** [Ubiquitous]
The `$schema` field shall reference the schema version in the form `"rubi-integration-rules/vX.Y"`.

**PIRF-F-005** [Ubiquitous]
The `rules` field shall be a JSON array of objects, each representing one integration rule.

**PIRF-F-006** [Ubiquitous]
Each rule object shall contain the mandatory fields `id` (unique integer), `pattern` (PIRF-Expr), `constraints` (array of PIRF-Expr), and `result` (PIRF-Expr).

**PIRF-F-007** [Ubiquitous]
Each rule object shall be allowed to contain the optional fields `description` (string), `section` (string), `references` (object), `comment` (string), and `derivation` (string).

**PIRF-F-008** [Ubiquitous]
The `id` field of each rule shall be a positive integer, unique across all loaded files.

### 5.2 Rule ordering and priority

**PIRF-F-009** [Ubiquitous]
Rule application priority shall be determined by **file loading order** as
specified in the load manifest (`meta.json` `"load_order"` array), followed
by position within each file's `rules` array. The first matching rule in
this combined sequence shall be applied.

> **Design note — Load-order priority.** RUBI's `Rubi.m` states: *"The order
> of loading the rule-files below is crucial to ensure a functional Rubi
> integrator!"* A rule in file `1.1.1.2` loaded before file `1.1.1.3` takes
> priority regardless of numeric id values. The `id` field serves only
> as a unique identifier for traceability and step-by-step display, not
> as a priority key.

**PIRF-F-009a** [Ubiquitous]
The `meta.json` file shall contain a `"load_order"` field: a JSON array of
file paths (relative to the rules root directory) specifying the exact
sequence in which rule files must be loaded. This array shall be authoritative
for rule priority.

**PIRF-F-009b** [Unwanted behaviour]
If a rule file exists in the directory tree but is not listed in the
`"load_order"` array, then the Loader shall emit a warning and shall not
load that file (orphan files are excluded by default).

**PIRF-F-009c** [Optional feature]
Where a rule file is listed in `"load_order"` with the prefix `"#"`,
it shall be treated as commented out (disabled) and shall not be loaded.
This mirrors RUBI's practice of commenting out rule sections (e.g.
`(*LoadRules[{"8 Special functions", "8.10 Bessel functions"}]; *)`).

### 5.3 File organisation

**PIRF-F-010** [Ubiquitous]
Rule files shall be organised in a directory tree reflecting the RUBI taxonomy (algebraic, exponential, logarithmic, trigonometric, etc.).

**PIRF-F-011** [Ubiquitous]
Each JSON file shall contain only the rules belonging to a single taxonomy section.

**PIRF-F-012** [Ubiquitous]
A `meta.json` file at the root of the directory tree shall describe the format
version, naming conventions, list of supported predicates, the load manifest
(`"load_order"`), conditional loading flags (`"feature_flags"`), and any
declared extensions.

**PIRF-F-012a** [Optional feature]
Where conditional loading is supported, the `meta.json` file shall contain
a `"feature_flags"` object mapping flag names to boolean defaults. Rule file
entries in `"load_order"` may include a `"requires"` field referencing a
feature flag, and shall only be loaded when the flag is true.

> **Design note — Conditional loading flags.**
> RUBI's `Rubi.m` defines `$LoadElementaryFunctionRules` and
> `$LoadShowSteps` as boolean flags controlling which rule sections are
> loaded. PIRF models this via `"feature_flags"` in `meta.json`:
> ```json
> {
>   "feature_flags": {
>     "load_elementary_function_rules": true,
>     "load_show_steps": true
>   },
>   "load_order": [
>     "1-algebraic/1.1-binomial/1.1.1-linear.json",
>     { "path": "4-trig/4.7-miscellaneous/4.7.5-inert-trig.json",
>       "requires": "load_elementary_function_rules" }
>   ]
> }
> ```

### 5.4 Size and performance

**PIRF-F-013** [Ubiquitous]
Each JSON file shall not exceed 1 MB to enable incremental loading.

**PIRF-F-014** [Ubiquitous]
The format shall support lazy loading by section, without requiring the full set of ~7,800 rules to be loaded at once.

### 5.5 Binary serialisation (CBOR)

> This subsection specifies an **optional** binary representation using CBOR
> (RFC 8949). The JSON format defined in §5.1–5.4 remains the **normative,
> canonical** format. CBOR is a derived cache format, analogous to `.pyc`
> files derived from `.py` sources.

#### 5.5.1 Format selection rationale

| Criterion | CBOR (RFC 8949) | MessagePack | Protocol Buffers | FlatBuffers |
|-----------|-----------------|-------------|-----------------|-------------|
| Formal standard | **IETF RFC** | Spec (no RFC) | Google (open) | Google (open) |
| JSON bijection | **Lossless** | Near-lossless | Lossy (names→field numbers) | Lossy |
| Recursive AST | **Native** (nested arrays) | Native | Requires `oneof` + recursive messages | No native recursion |
| Schema requirement | **None** (schemaless) | None | Mandatory `.proto` | Mandatory `.fbs` |
| Zero-copy read | No | No | No | **Yes** |
| Typical size vs JSON | **60–70%** | 60–70% | 40–50% | 30–40% |
| Parse speed vs JSON | **2–5× faster** | 2–5× faster | 5–10× faster | Near-instant (zero-copy) |

CBOR is selected because it preserves the exact same data model as JSON
(arrays, maps, strings, numbers), requires no external schema file, is an
IETF standard, and supports lossless round-trip conversion with JSON.

#### 5.5.2 Architecture

```
Source of truth          Derived cache          Optional target
─────────────────       ──────────────         ────────────────
  *.json (§5.1)  ──▶  pirf-compile   ──▶  *.pirf.cbor (§5.5)
  (normative,          (CLI tool)          (binary, fast load,
   git-versioned,                           NOT versioned in git)
   human-readable)
                                     ──▶  *.pirf.fbs (future)
                                           (zero-copy, WASM)
```

#### 5.5.3 Requirements

**PIRF-B-001** [Optional feature]
Where binary serialisation is supported, the system shall use CBOR
(Concise Binary Object Representation, IETF RFC 8949) as the binary format.

**PIRF-B-002** [Ubiquitous]
The JSON format (§5.1–5.4) shall remain the normative, canonical format.
A CBOR file shall never be the sole representation of a rule set.

**PIRF-B-003** [Ubiquitous]
A CBOR-encoded file shall be **semantically identical** to its JSON
source: converting JSON → CBOR → JSON shall produce a byte-identical result
(after JSON canonical normalisation as per RFC 8785).

**PIRF-B-004** [Ubiquitous]
CBOR-encoded files shall use the file extension `.pirf.cbor`.

**PIRF-B-005** [Ubiquitous]
CBOR-encoded files shall use CBOR Diagnostic Notation tag
`55799` (self-described CBOR, magic bytes `0xD9D9F7`) as first item
to enable reliable format detection.

**PIRF-B-006** [Event-driven]
When the `pirf-compile` tool receives a directory of JSON rule files,
it shall produce one `.pirf.cbor` file per JSON source file, preserving
the directory structure.

**PIRF-B-007** [Optional feature]
Where a single-file archive is preferred, the `pirf-compile` tool shall
produce a single `.pirf.cbor` file containing a CBOR map with section
paths as keys and rule arrays as values.

**PIRF-B-008** [Event-driven]
When the Loader detects a `.pirf.cbor` file alongside its `.json` source,
the Loader shall compare modification timestamps and load the newer file.

**PIRF-B-009** [Unwanted behaviour]
If a `.pirf.cbor` file is older than its `.json` source, then the Loader
shall ignore the CBOR file, load the JSON source, and emit a warning
recommending recompilation.

**PIRF-B-010** [Unwanted behaviour]
If a `.pirf.cbor` file fails integrity validation (truncated, corrupted,
or schema-version mismatch), then the Loader shall fall back to the JSON
source and emit a warning.

**PIRF-B-011** [Event-driven]
When the `pirf-compile` tool produces a `.pirf.cbor` file, it shall embed
a CBOR map header containing: `pirf_version` (string), `source_sha256`
(hex string of the JSON source SHA-256), `compiled_at` (ISO 8601 timestamp),
and `rule_count` (integer).

**PIRF-B-012** [Ubiquitous]
The CBOR encoding shall use CBOR definite-length arrays and maps
(not indefinite-length) to enable size-prefixed reads and memory pre-allocation.

**PIRF-B-013** [Optional feature]
Where streaming loading is supported, the `pirf-compile` tool shall optionally
produce CBOR Sequences (RFC 8742) where each top-level item is one rule,
enabling incremental parsing without loading the full file.

**PIRF-B-014** [Ubiquitous]
PIRF-Expr expressions within CBOR files shall use the same array-based
representation as in JSON: `["Add", 1, "x"]` becomes a CBOR array of
three items (text string, integer, text string). No CBOR-specific encoding
(e.g. CBOR tags for operators) shall be used, to preserve JSON bijection.

**PIRF-B-015** [Optional feature]
Where string interning is supported, the `pirf-compile` tool shall use
CBOR shared string references (RFC 8949 §3.4.6, tag 25/29) to deduplicate
repeated operator names (`"Add"`, `"Multiply"`, `"FreeQ"`, etc.) across
the file, reducing size by an estimated additional 15–25%.

#### 5.5.4 Performance targets

**PIRF-B-020** [Ubiquitous]
Loading the full rule set (~7,800 rules) from CBOR cache shall complete
in under **1 second** on a standard workstation (compared to 5 seconds
for JSON, per PIRF-L-020).

**PIRF-B-021** [Ubiquitous]
The CBOR representation of the full rule set shall not exceed **70%** of
the JSON representation size.

**PIRF-B-022** [Optional feature]
Where CBOR + string interning (PIRF-B-015) is used, the representation
shall not exceed **50%** of the JSON representation size.

#### 5.5.5 Future binary formats

**PIRF-B-030** [Optional feature]
Where zero-copy loading is required (e.g. WebAssembly, embedded systems),
an implementation may provide FlatBuffers (`.pirf.fbs`) serialisation
as an additional derived format, provided:
  (a) it is generated from the JSON source via `pirf-compile`,
  (b) it preserves semantic identity with the JSON source,
  (c) it is documented as a non-normative acceleration format.

**PIRF-B-031** [Ubiquitous]
No binary format shall introduce information not present in the JSON source.
Binary formats are **derived**, never **authoritative**.

---

## 6. Requirements — Wildcards and pattern matching

**PIRF-W-001** [Ubiquitous]
A mandatory wildcard shall be denoted by an underscore suffix: `"x_"`, `"m_"`, `"n_"`.

**PIRF-W-002** [Ubiquitous]
An optional wildcard with a default value shall be denoted by a dot suffix
followed by an optional default specifier: `"a."` (default 0), `"b."` (default 1),
or `"m.3"` (default 3).

> **Design note.** RUBI uses Mathematica's `Optional` with arbitrary
> defaults, not just 0 or 1. The `WC('m', S(1))` pattern in the SymPy port
> confirms this. The dot-suffix grammar supports explicit integer defaults.

**PIRF-W-002a** [Ubiquitous]
If no default specifier follows the dot suffix, the default shall be determined
by the wildcard name: names starting with additive-role letters (`a`, `b`, `c`, `d`,
`e`, `f`, `g`, `h`) default to 0; names starting with multiplicative-role letters
(`m`, `n`, `p`, `q`) default to 1. This convention matches RUBI's most common
usage of `Optional` defaults.

**PIRF-W-003** [Ubiquitous]
A typed wildcard shall be denoted by an underscore suffix followed by the type: `"m_integer"`, `"p_rational"`, `"n_positive"`.

**PIRF-W-004** [Ubiquitous]
The Engine shall treat wildcards as binding variables captured when matching against the integrand.

**PIRF-W-005** [Ubiquitous]
Two occurrences of the same wildcard name within a pattern shall match the same sub-expression.

**PIRF-W-006** [Ubiquitous]
The format shall support sequence wildcards for matching variable-length
argument lists:
- `"xs__"` (double underscore) — matches one or more sub-expressions (BlankSequence)
- `"xs___"` (triple underscore) — matches zero or more sub-expressions (BlankNullSequence)

> **Design note.** Some RUBI rules use Mathematica's `__` and `___` patterns
> for matching variable-length sequences in sums and products. Although most
> rules use fixed-arity patterns, sequence wildcards are needed for full
> coverage of miscellaneous and normalization rules.

---

## 7. Requirements — Constraints (predicates)

### 7.1 Independence and free-variable predicates

**PIRF-C-001** [Ubiquitous]
The format shall support the predicate `["FreeQ", var, x]` meaning `var` is free of `x`.

**PIRF-C-002** [Ubiquitous]
The format shall support the predicate `["FreeQ", ["List", v1, v2, ...], x]` meaning all listed variables are free of `x`.

**PIRF-C-003** [Ubiquitous]
The format shall support `["IndependentQ", u, x]` meaning `u` is independent of `x` (stricter than FreeQ).

### 7.2 Type predicates

**PIRF-C-010** [Ubiquitous]
The format shall support the following **type predicates**:

| Predicate | Semantics |
|-----------|-----------|
| `IntegerQ` | Is an integer |
| `PositiveIntegerQ` | Is a positive integer |
| `NegativeIntegerQ` | Is a negative integer |
| `FractionQ` | Is an explicit fraction |
| `RationalQ` | Is rational (integer or fraction) |
| `ComplexNumberQ` | Is an explicit complex number |
| `RealNumericQ` | Is a real numeric value |
| `SqrtNumberQ` | Is equivalent to sqrt of a rational |
| `SqrtNumberSumQ` | Is a sum of sqrt-numbers |
| `FractionOrNegativeQ` | Is a fraction or negative integer |
| `FractionalPowerOfSquareQ` | Is a fractional power of a perfect square |

### 7.3 Integer comparison predicates

**PIRF-C-011** [Ubiquitous]
The format shall support the following **integer comparison predicates** (RUBI-specific, combining integer test with comparison):

| Predicate | Arity | Semantics |
|-----------|-------|-----------|
| `IGtQ` | 2 | u is integer AND u > n |
| `ILtQ` | 2 | u is integer AND u < n |
| `IGeQ` | 2 | u is integer AND u ≥ n |
| `ILeQ` | 2 | u is integer AND u ≤ n |
| `IntegersQ` | 1+ | All arguments are integers |

### 7.4 General comparison predicates

**PIRF-C-012** [Ubiquitous]
The format shall support the comparison predicates: `EqQ`, `NeQ`, `GtQ`, `LtQ`, `GeQ`, `LeQ`.

### 7.5 Algebraic predicates

**PIRF-C-013** [Ubiquitous]
The format shall support the algebraic predicates: `ZeroQ`, `NonzeroQ`, `PositiveQ`, `NegativeQ`.

**PIRF-C-014** [Ubiquitous]
The format shall support the combinatorial predicates: `OddQ`, `EvenQ`.

### 7.6 Structural predicates

**PIRF-C-020** [Ubiquitous]
The format shall support the following **structural predicates**:

| Predicate | Arity | Semantics |
|-----------|-------|-----------|
| `PolynomialQ` | 2 | expr is a polynomial in x |
| `LinearQ` | 2 | expr is linear in x |
| `QuadraticQ` | 2 | expr is quadratic in x |
| `BinomialQ` | 2 | expr is a binomial in x |
| `TrinomialQ` | 2 | expr is a trinomial in x |
| `PowerQ` | 1 | expr has the form base^exponent |
| `SumQ` | 1 | expr is a sum (head is Plus/Add) |
| `ProductQ` | 1 | expr is a product (head is Times/Multiply) |
| `IntegerPowerQ` | 1 | expr has the form base^integer |
| `FractionalPowerQ` | 1 | expr has the form base^fraction |
| `QuotientOfLinearsQ` | 2 | expr is (a+bx)/(c+dx) in x |

### 7.7 Function-type predicates

**PIRF-C-021** [Ubiquitous]
The format shall support the following **function-type predicates**:

| Predicate | Semantics |
|-----------|-----------|
| `TrigQ` | expr is a circular trig function call |
| `HyperbolicQ` | expr is a hyperbolic function call |
| `InverseTrigQ` | expr is an inverse circular trig function call |
| `InverseHyperbolicQ` | expr is an inverse hyperbolic function call |
| `LogQ` | expr is a Log call |
| `InverseFunctionQ` | expr is any inverse function (log, inverse trig/hyp) |
| `ElementaryFunctionQ` | expr is an elementary function |
| `AlgebraicFunctionQ` | expr is algebraic in x |
| `AlgebraicTrigFunctionQ` | expr is algebraic in trig functions of x |
| `RationalFunctionQ` | expr is a rational function of x |
| `BinomialMatchQ` | expr matches a binomial form in x |

### 7.8 Calculus-aware predicates

**PIRF-C-022** [Ubiquitous]
The format shall support the following **calculus-aware predicates**:

| Predicate | Arity | Semantics |
|-----------|-------|-----------|
| `CalculusQ` | 1 | expr involves D, Integrate, Sum, etc. |
| `CalculusFreeQ` | 2 | expr has no calculus functions in x |
| `IntegralFreeQ` | 1 | expr has no residual Int/Unintegrable operators |
| `TrigHyperbolicFreeQ` | 2 | expr is free of trig/hyp functions in x |
| `FunctionOfTrigOfLinearQ` | 2 | expr is algebraic in trig(a+bx) |

### 7.9 Function-of predicates

**PIRF-C-023** [Ubiquitous]
The format shall support the following **function-of predicates** used by RUBI
to detect substitution opportunities:

| Predicate | Arity | Semantics |
|-----------|-------|-----------|
| `FunctionOfQ` | 3 | u is a function of v w.r.t. x |
| `FunctionOfExponentialQ` | 2 | u is a function of F^(linear in x) |
| `FunctionOfExponentialTest` | 2 | Test for exponential substitution (sets fluid variables) |
| `PureFunctionOfSinQ` | 3 | u is a pure function of Sin[v] and/or Csc[v] |
| `PureFunctionOfCosQ` | 3 | u is a pure function of Cos[v] and/or Sec[v] |
| `PureFunctionOfTanQ` | 3 | u is a pure function of Tan[v] and/or Cot[v] |
| `PureFunctionOfCotQ` | 3 | u is a pure function of Cot[v] and/or Tan[v] |
| `FunctionOfSinQ` | 3 | u involves Sin[v] (not necessarily pure) |
| `FunctionOfCosQ` | 3 | u involves Cos[v] |
| `FunctionOfTanQ` | 3 | u involves Tan[v] |
| `FunctionOfSinhQ` | 3 | u involves Sinh[v] |
| `FunctionOfCoshQ` | 3 | u involves Cosh[v] |
| `FunctionOfTanhQ` | 3 | u involves Tanh[v] |
| `FunctionOfHyperbolicQ` | 2 | u is a function of hyp(linear in x), returns v or False |
| `FunctionOfTrigQ` | 3 | u is a function of trig(v) w.r.t. x |
| `FractionalPowerSubexpressionQ` | 3 | u contains (v)^(1/n) type sub-expressions |

> **Design note.** These predicates are central to RUBI's substitution
> strategy. They appear frequently in constraints and some (`FunctionOfExponentialTest`)
> set fluid variables (`$base$`, `$expon$`) as side effects. The Phrasebook
> must implement these with the correct semantics including any side effects.

### 7.10 Logical connectives

**PIRF-C-030** [Ubiquitous]
The format shall support `["And", c1, c2, ...]`, `["Or", c1, c2, ...]`, and `["Not", c]`.

**PIRF-C-031** [Event-driven]
When the `constraints` array contains multiple predicates, the Loader shall interpret them as an implicit conjunction (AND).

### 7.11 Extensibility

**PIRF-C-040** [Optional feature]
Where a target system supports additional predicates, the format shall allow registration of custom predicates via `meta.json`, in the `"custom_predicates"` field.

**PIRF-C-041** [Unwanted behaviour]
If a constraint references an unrecognised predicate, then the Loader shall skip the rule and emit a warning without interrupting loading.

---

## 8. Requirements — Rule taxonomy

**PIRF-T-001** [Ubiquitous]
The taxonomy shall follow the **actual RUBI hierarchy** with 9 main categories,
matching the directory structure in the RUBI IntegrationRules repository:

| Section | Title | Subsections |
|---------|-------|-------------|
| 1 | Algebraic functions | 1.1 Binomial products (1.1.1 Linear, 1.1.2 Quadratic, 1.1.3 General, 1.1.4 Improper), 1.2 Trinomial products (1.2.1 Quadratic, 1.2.2 Quartic, 1.2.3 General, 1.2.4 Improper), 1.3 Miscellaneous |
| 2 | Exponentials | 2.1–2.3 |
| 3 | **Logarithms** | 3.1–3.5 |
| 4 | Trig functions | 4.1 Sine, 4.3 Tangent, 4.5 Secant, 4.7 Miscellaneous |
| 5 | Inverse trig functions | 5.1 Inverse sine, 5.3 Inverse tangent, 5.5 Inverse secant |
| 6 | Hyperbolic functions | 6.1 Hyp. sine, 6.3 Hyp. tangent, 6.5 Hyp. secant, 6.7 Miscellaneous |
| 7 | Inverse hyperbolic functions | 7.1 Inv. hyp. sine, 7.3 Inv. hyp. tangent, 7.5 Inv. hyp. secant |
| 8 | Special functions | 8.1 Error, 8.2 Fresnel, 8.3 Exp. integral, 8.4 Trig integral, 8.5 Hyp. integral, 8.6 Gamma, 8.7 Zeta, 8.8 Polylogarithm, 8.9 Product log, 8.10 Bessel |
| 9 | Miscellaneous | 9.1 Integrand simplification, 9.2 Derivative integration, 9.3 Piecewise linear, 9.4 Miscellaneous integration |

> **Design note — Taxonomy.** This taxonomy matches RUBI's actual directory
> structure in the IntegrationRules repository. Note that **Logarithms are
> section 3** (not Trig), followed by Trig at 4, Inverse trig at 5,
> Hyperbolic at 6, Inverse hyperbolic at 7, Special at 8, and
> Miscellaneous at 9.

**PIRF-T-001a** [Ubiquitous]
Note that RUBI uses **even-numbered subsections for the co-function rules**
that are handled by normalization to the odd-numbered form. For example,
section 4 (Trig) has subsections 4.1 (Sine), 4.3 (Tangent), 4.5 (Secant),
and 4.7 (Miscellaneous) — there are no 4.2, 4.4, or 4.6 subsections because
Cosine, Cotangent, and Cosecant rules are normalized to their co-function
equivalents. The file path structure shall preserve these gaps.

**PIRF-T-002** [Ubiquitous]
Each section shall be subdivided according to the RUBI structure (e.g. 1.1 = binomial products, 1.1.1 = linear binomials, etc.).

**PIRF-T-003** [Ubiquitous]
The section number of a JSON file shall correspond to the file path within the directory tree.

**PIRF-T-004** [Optional feature]
Where a target system defines additional categories, the format shall allow custom sections numbered from 10 onwards.

---

## 9. Requirements — Loader

### 9.1 Loading

**PIRF-L-001** [Event-driven]
When the Loader receives a path to a JSON file, the Loader shall validate the file structure against the JSON Schema before loading.

**PIRF-L-002** [Event-driven]
When the Loader receives a path to a directory containing `meta.json`, the
Loader shall read the `"load_order"` manifest and load rule files in the
specified sequence.

> **Design note.** RUBI requires a specific load order that is not purely
> alphabetical. The `"load_order"` manifest is authoritative.

**PIRF-L-003** [Event-driven]
When the Loader loads a file, the Loader shall parse each rule and convert it to the target engine's internal representation via the Phrasebook.

**PIRF-L-004** [Event-driven]
When loading is complete, the Loader shall preserve rules in **load-manifest
order**, not sorted by `id`. The `id` field is for identification and
traceability only, not for priority.

### 9.2 Validation

**PIRF-L-010** [Event-driven]
When the Loader encounters malformed JSON, the Loader shall reject the file with an error indicating filename and error position.

**PIRF-L-011** [Unwanted behaviour]
If a file contains an incompatible `$schema` version, then the Loader shall warn and attempt loading in degraded mode.

**PIRF-L-012** [Unwanted behaviour]
If two rules share the same `id`, then the Loader shall emit an error and refuse to complete loading.

**PIRF-L-013** [Unwanted behaviour]
If a PIRF-Expr contains an unrecognised operator not declared in extensions, then the Loader shall warn and skip the affected rule.

### 9.3 Performance

**PIRF-L-020** [Ubiquitous]
The Loader shall load the full rule set (~7,800 rules) in under 5 seconds on a standard workstation.

**PIRF-L-021** [Optional feature]
Where lazy loading is supported, the Loader shall load only requested sections on demand.

**PIRF-L-022** [Optional feature]
Where caching is supported, the Loader shall serialise loaded rules to a native binary format for faster subsequent loads.

---

## 10. Requirements — Integration engine

### 10.1 Pattern matching

**PIRF-E-001** [Event-driven]
When the Engine receives an expression to integrate, the Engine shall traverse rules in load-manifest priority order and apply the first matching rule.

**PIRF-E-002** [Event-driven]
When a pattern contains wildcards, the Engine shall bind each wildcard to a sub-expression, respecting typed constraints.

**PIRF-E-003** [Event-driven]
When a match is found, the Engine shall instantiate the result by replacing wildcards with bound values.

**PIRF-E-004** [Unwanted behaviour]
If no rule matches, then the Engine shall return the integral in unevaluated form `Int[expr, x]`.

### 10.2 Recursion

**PIRF-E-010** [Event-driven]
When the result contains `["Int", expr, x]`, the Engine shall re-invoke pattern matching on `expr`.

**PIRF-E-011** [Event-driven]
When the result contains `["Dist", c, expr, x]`, the Engine shall compute `c · Int[expr, x]`.

**PIRF-E-012** [Event-driven]
When the result contains `["Subst", expr, x, u]`, the Engine shall substitute `x` by `u` in `expr`.

### 10.3 Recursion safety

**PIRF-E-020** [Ubiquitous]
The Engine shall enforce a configurable maximum recursion depth (default: 100).

**PIRF-E-021** [Unwanted behaviour]
If recursion exceeds the limit, then the Engine shall halt and return the integral in unevaluated form.

**PIRF-E-022** [Unwanted behaviour]
If the same rule is applied to the same expression on two consecutive invocations, then the Engine shall break the loop.

### 10.4 Simplification

**PIRF-E-030** [Event-driven]
When the result contains `["Simp", expr, x]`, the Engine shall apply algebraic simplifications.

**PIRF-E-031** [Event-driven]
When the final result contains no residual `Int` operators, the Engine shall apply a global simplification pass.

### 10.5 Inert function handling

**PIRF-E-040** [Event-driven]
When the Engine begins processing a trigonometric or hyperbolic integrand,
the Engine shall first apply `DeactivateTrig` to convert active trig/hyp
function calls to inert forms, preventing premature CAS evaluation during
pattern matching.

**PIRF-E-041** [Event-driven]
When the Engine produces a final result with no residual `Int` operators,
the Engine shall apply `ActivateTrig` to convert any remaining inert forms
back to active (evaluable) forms.

> **Design note.** This mechanism is essential to RUBI's correctness.
> Without inert forms, the CAS may simplify `Sin[x]^2 + Cos[x]^2` to `1`
> before pattern matching can identify the appropriate rule. RUBI sections
> 4.7.5 and 4.7.9 contain the normalization rules for this purpose.

---

## 11. Requirements — Step-by-step display

**PIRF-S-001** [Optional feature]
Where step-by-step display is supported, the Engine shall record rule `id`, before-expression, and after-expression for each application.

**PIRF-S-002** [Complex: While + When]
While step-by-step mode is enabled, when the Engine applies a rule, the Engine shall emit an event with rule `id`, description, and intermediate result.

**PIRF-S-003** [Optional feature]
Where statistics are supported, the Engine shall compute: step count, distinct rule count, input/output leaf count, and rules-to-size ratio.

---

## 12. Requirements — Test suite

**PIRF-V-001** [Ubiquitous]
The format shall include a test suite as JSON files separate from rule files.

**PIRF-V-002** [Ubiquitous]
Each test problem shall contain: `id`, `integrand` (PIRF-Expr), `variable` (string), `optimal_antiderivative` (PIRF-Expr), and `num_steps` (integer).

**PIRF-V-003** [Event-driven]
When a test is executed, the result shall be compared to `optimal_antiderivative` by symbolic equivalence (not syntactic equality).

**PIRF-V-004** [Event-driven]
When the result differs, the validator shall grade per RUBI scale: A (≤ 2× optimal), B (> 2× but correct), C (correct, sub-optimal), F (incorrect/timeout).

**PIRF-V-005** [Ubiquitous]
The test suite shall cover a minimum of 72,000 problems.

**PIRF-V-006** [Unwanted behaviour]
If the Engine produces no result within configurable timeout (default: 120s), then the validator shall grade F.

---

## 13. Requirements — Portability and interoperability

### 13.1 Language independence

**PIRF-P-001** [Ubiquitous]
The format shall be independent of any programming language — no rule file shall contain executable code.

**PIRF-P-002** [Ubiquitous]
The format shall use exclusively native JSON data types.

**PIRF-P-003** [Ubiquitous]
The format shall be parsable by any standard JSON parser without extensions.

### 13.2 RUBI compatibility

**PIRF-P-010** [Ubiquitous]
The format shall represent the entirety of RUBI's rule set (~7,800 rules) without loss of information.

**PIRF-P-011** [Ubiquitous]
The format shall preserve RUBI rule numbering (`id` field) for traceability.

**PIRF-P-012** [Ubiquitous]
The format shall represent all RUBI predicates (FreeQ, IntegerQ, IGtQ, NonzeroQ, PolynomialQ, TrigQ, FunctionOfQ, FunctionOfExponentialQ, etc.).

**PIRF-P-013** [Ubiquitous]
The format shall represent all RUBI utility functions (Int, Dist, Subst, Simp, ExpandIntegrand, NormalizeIntegrand, SimplifyAntiderivative, Rt, ActivateTrig, DeactivateTrig, etc.).

**PIRF-P-014** [Ubiquitous]
The format shall represent all mathematical functions appearing in RUBI rule results, including special functions (PolyLog, EllipticF, Erf, Gamma, Hypergeometric2F1, BesselJ, etc.).

**PIRF-P-015** [Ubiquitous]
The format shall preserve the exact rule loading order defined in RUBI's
`Rubi.m` via the `"load_order"` manifest in `meta.json`.

### 13.3 Phrasebook architecture

**PIRF-P-020** [Ubiquitous]
Each host CAS integration shall provide a **Phrasebook** — an adapter module translating between PIRF-Expr and the host CAS internal representation.

**PIRF-P-021** [Ubiquitous]
The Phrasebook shall be the **only** component with host-CAS-specific code; the Loader, rule files, and test suite shall remain host-agnostic.

**PIRF-P-022** [Ubiquitous]
The Phrasebook shall implement all utility functions defined in §4.11 using the host CAS's native capabilities.

**PIRF-P-022a** [Ubiquitous]
The Phrasebook shall implement the inert/active trig function mechanism
(§4.11.3, `ActivateTrig`/`DeactivateTrig`) appropriate to the host CAS.
In CAS environments that do not auto-simplify (e.g. a Lisp-based symbolic
engine), this may be a no-op.

**PIRF-P-023** [Optional feature]
Where an OpenMath Phrasebook is provided, it shall map PIRF-Expr operators to OpenMath CD symbols per Annex A.

**PIRF-P-024** [Optional feature]
Where a Content MathML Phrasebook is provided, it shall map PIRF-Expr operators to Content MathML elements per Annex A.

**PIRF-P-025** [Optional feature]
Where a MathLive MathJSON Phrasebook is provided, the mapping shall be identity for all core arithmetic and direct trigonometric operators, and shall translate inverse trigonometric operators (`Asin`→`["Apply", ["InverseFunction", "Sin"], x]`, etc.) and integration-specific operators (`Int`, `Dist`, `Subst`, `Simp`) to appropriate MathJSON representations or opaque symbols.

### 13.4 Extensibility

**PIRF-P-030** [Optional feature]
Where a target system defines additional operators, the format shall allow their declaration in `meta.json` `"extensions"` without schema modification.

**PIRF-P-031** [Optional feature]
Where a target system defines custom predicates, the format shall allow their declaration in `meta.json` `"custom_predicates"`.

**PIRF-P-032** [Optional feature]
Where an automatic converter from Mathematica `.m` / `.nb` files is available, the converter shall produce valid JSON conforming to this specification.

---

## 14. Requirements — Licensing and documentation

**PIRF-D-001** [Ubiquitous]
The format shall be documented by a validatable JSON Schema (draft-07 or later).

**PIRF-D-002** [Ubiquitous]
Rule files converted from RUBI shall cite the RUBI Software License (MIT) in `meta.json` `"license"`.

**PIRF-D-003** [Ubiquitous]
Each rule shall be able to reference bibliographic sources (G&R, CRC, A&S) via the optional `references` field.

---

## 15. Non-functional requirements

**PIRF-N-001** [Ubiquitous]
The schema shall be versioned per SemVer 2.0.0.

**PIRF-N-002** [Ubiquitous]
Backwards-incompatible changes shall increment the major version.

**PIRF-N-003** [Ubiquitous]
The format shall be validatable by standard JSON Schema tooling (ajv, jsonschema, etc.).

**PIRF-N-004** [Ubiquitous]
The format shall support annotations via `description` or `comment` fields.

---

## 16. Traceability matrix

| Category | Identifiers | Count |
|----------|------------|-------|
| PIRF-Expr language (X) | PIRF-X-001 to X-043, X-031a | 27 |
| File format (F) | PIRF-F-001 to F-014, F-009a/b/c, F-012a | 18 |
| Binary serialisation (B) | PIRF-B-001 to B-031 | 20 |
| Wildcards (W) | PIRF-W-001 to W-006, W-002a | 7 |
| Constraints (C) | PIRF-C-001 to C-041, C-023 | 23 |
| Taxonomy (T) | PIRF-T-001 to T-004, T-001a | 5 |
| Loader (L) | PIRF-L-001 to L-022 | 11 |
| Engine (E) | PIRF-E-001 to E-041 | 14 |
| Step-by-step (S) | PIRF-S-001 to S-003 | 3 |
| Validation (V) | PIRF-V-001 to V-006 | 6 |
| Portability (P) | PIRF-P-001 to P-032, P-015, P-022a | 18 |
| Documentation (D) | PIRF-D-001 to D-003 | 3 |
| Non-functional (N) | PIRF-N-001 to N-004 | 4 |
| **Total** | | **159** |

---

## 17. EARS pattern distribution

| EARS pattern | Count | % |
|-------------|-------|---|
| Ubiquitous | 100 | 63% |
| Event-driven (When) | 21 | 13% |
| Unwanted behaviour (If…then) | 12 | 8% |
| Optional feature (Where) | 25 | 16% |
| Complex (While…When) | 1 | 1% |
| State-driven (While) | 0 | 0% |

---

## Annex A — Operator mapping tables (normative)

### A.1 Core arithmetic

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML |
|-----------|-------------------|-------------------|----------------|
| `Add` | `Add` | `arith1#plus` | `<plus/>` |
| `Subtract` | `Subtract` | `arith1#minus` | `<minus/>` (binary) |
| `Multiply` | `Multiply` | `arith1#times` | `<times/>` |
| `Divide` | `Divide` | `arith1#divide` | `<divide/>` |
| `Power` | `Power` | `arith1#power` | `<power/>` |
| `Negate` | `Negate` | `arith1#unary_minus` | `<minus/>` (unary) |
| `Sqrt` | `Sqrt` | `arith1#root` (n=2) | `<root/>` |
| `Abs` | `Abs` | `arith1#abs` | `<abs/>` |
| `Factorial` | `Factorial` | `integer1#factorial` | `<factorial/>` |

### A.2 Trigonometric functions

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML |
|-----------|-------------------|-------------------|----------------|
| `Sin` | `Sin` | `transc1#sin` | `<sin/>` |
| `Cos` | `Cos` | `transc1#cos` | `<cos/>` |
| `Tan` | `Tan` | `transc1#tan` | `<tan/>` |
| `Cot` | `Cot` | `transc1#cot` | `<cot/>` |
| `Sec` | `Sec` | `transc1#sec` | `<sec/>` |
| `Csc` | `Csc` | `transc1#csc` | `<csc/>` |

### A.3 Hyperbolic functions

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML |
|-----------|-------------------|-------------------|----------------|
| `Sinh` | `Sinh` | `transc1#sinh` | `<sinh/>` |
| `Cosh` | `Cosh` | `transc1#cosh` | `<cosh/>` |
| `Tanh` | `Tanh` | `transc1#tanh` | `<tanh/>` |
| `Coth` | `Coth` | `transc1#coth` | `<coth/>` |
| `Sech` | `Sech` | `transc1#sech` | `<sech/>` |
| `Csch` | `Csch` | `transc1#csch` | `<csch/>` |

### A.4 Inverse trigonometric functions

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML | Notes |
|-----------|-------------------|-------------------|----------------|-------|
| `Asin` | `["Apply", ["InverseFunction", "Sin"], x]` | `transc1#arcsin` | `<arcsin/>` | **Divergence** ¹ |
| `Acos` | `["Apply", ["InverseFunction", "Cos"], x]` | `transc1#arccos` | `<arccos/>` | **Divergence** ¹ |
| `Atan` | `["Apply", ["InverseFunction", "Tan"], x]` | `transc1#arctan` | `<arctan/>` | **Divergence** ¹ |
| `Acot` | `["Apply", ["InverseFunction", "Cot"], x]` | `transc1#arccot` | `<arccot/>` | **Divergence** ¹ |
| `Asec` | `["Apply", ["InverseFunction", "Sec"], x]` | `transc1#arcsec` | `<arcsec/>` | **Divergence** ¹ |
| `Acsc` | `["Apply", ["InverseFunction", "Csc"], x]` | `transc1#arccsc` | `<arccsc/>` | **Divergence** ¹ |

> **¹ Inverse trigonometric divergence.**
> MathLive MathJSON does **not** define `Arcsin` etc. as first-class operators.
> PIRF uses atomic operators for the reasons stated in §4.5.
> **Phrasebook rule:** `["Asin", x]` ↔ `["Apply", ["InverseFunction", "Sin"], x]`.

### A.5 Inverse hyperbolic functions

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML | Notes |
|-----------|-------------------|-------------------|----------------|-------|
| `Asinh` | compositional | `transc1#arcsinh` | `<arcsinh/>` | **Divergence** ¹ |
| `Acosh` | compositional | `transc1#arccosh` | `<arccosh/>` | **Divergence** ¹ |
| `Atanh` | compositional | `transc1#arctanh` | `<arctanh/>` | **Divergence** ¹ |
| `Acoth` | compositional | `transc1#arccoth` | `<arccoth/>` | **Divergence** ¹ |
| `Asech` | compositional | `transc1#arcsech` | `<arcsech/>` | **Divergence** ¹ |
| `Acsch` | compositional | `transc1#arccsch` | `<arccsch/>` | **Divergence** ¹ |

### A.6 Exponential and logarithmic

| PIRF-Expr | MathLive MathJSON | OpenMath CD#symbol | Content MathML |
|-----------|-------------------|-------------------|----------------|
| `Exp` | `Exp` | `transc1#exp` | `<exp/>` |
| `Log` | `Log` | `transc1#ln` | `<ln/>` |

### A.7 Integration-specific operators (PIRF-only)

| PIRF-Expr | Semantics | OpenMath equivalent | MathML equivalent |
|-----------|-----------|--------------------|--------------------|
| `Int` | ∫ $1 d$2 | `calculus1#int` (partial) | `<int/>` (partial) |
| `Dist` | $1 · ∫ $2 d$3 | *(none)* | *(none)* |
| `Subst` | substitution | *(none)* | *(none)* |
| `Simp` | simplification | *(none)* | *(none)* |
| `ExpandIntegrand` | expand before ∫ | *(none)* | *(none)* |
| `Unintegrable` | not integrable in closed form | *(none)* | *(none)* |
| `CannotIntegrate` | engine cannot integrate | *(none)* | *(none)* |

> These operators are unique to rule-based integration systems.
> No existing standard provides equivalents. This specification defines them authoritatively.