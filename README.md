# HELICON

### **H**yperbolic · **E**uclidean · **L**inear **I**terative **C**ORDIC **O**ption **N**etwork

> **A curvature-ladder architecture for deterministic option risk.**
> One traversal. Every Greek. Precision allocated by the Greeks themselves.

---

## 0 · The Convair Problem, Restated

In June 1956, an engineer at Convair's aeroelectronics group filed an internal report numbered IAR-1.148. He had a specific problem: the B-58 Hustler — the first bomber built to hold Mach 2 — was navigating with an analog resolver, and an analog resolver could not keep up with an airframe moving faster than the sound of its own engines. The airplane needed to know where it was, continuously, and the mathematics of knowing where you are is the mathematics of rotating a coordinate frame.

He found his answer in a 1946 edition of the *CRC Handbook of Chemistry and Physics* — a trigonometric identity sitting in a reference table, uncelebrated. The identity said that a rotation by an awkward angle could be decomposed into a sequence of rotations by angles whose tangents are exact negative powers of two. And a multiplication by a power of two, on a binary machine, is not a multiplication. It is a wire.

The technique was, in the author's own retrospective phrase, **born out of necessity**. It published in 1959 as *The CORDIC Trigonometric Computing Technique*. Fifteen years later, when Hewlett-Packard needed transcendental functions inside a calculator that fit in a shirt pocket, the same idea was generalized into three geometries at once — circular, linear, hyperbolic — under the title *A unified algorithm for elementary functions*. It went into the HP-9100 and the HP-35. It went into the Intel 80x87 coprocessor family. It went into the Motorola 68881. Somewhere in the 1990s it stopped being a headline and became plumbing.

Here is the part worth noticing.

The B-58 problem and the options problem are **the same problem**. Both ask a machine to answer a geometric question about *where a system is right now*, under a clock that does not negotiate, with a wrong answer being worse than a late answer only in the narrowest sense — because in both domains a late answer *is* a wrong answer. The airplane is at Mach 2. The quote is at 53 million messages per second.

Nobody wrote down the correspondence, because the two fields did not share conferences. HELICON is that correspondence, written down and made load-bearing.

---

## 1 · The Curvature Ladder

The unified formulation carries a single parameter, conventionally written **m**, that selects the geometry of the plane in which rotation occurs:

| m | Geometry | Invariant | Micro-angle | Native outputs |
|:--|:--|:--|:--|:--|
| **+1** | Circular (Euclidean) | x² + y² | arctan(2⁻ⁱ) | sin, cos, arctan, magnitude |
| **0** | Linear (degenerate) | x | 2⁻ⁱ | multiply, divide |
| **−1** | Hyperbolic | x² − y² | artanh(2⁻ⁱ) | sinh, cosh, artanh, exp, ln, √ |

Three geometries, one datapath. The recurrences differ only in a sign and a table:

```
x[i+1] = x[i] − m · δ[i] · y[i] · 2^(−i)
y[i+1] = y[i] +     δ[i] · x[i] · 2^(−i)
z[i+1] = z[i] −     δ[i] · α(m, i)
```

`δ[i] ∈ {−1, +1}` is a **sign decision**, not an arithmetic choice. `2^(−i)` is a **barrel shift**, not a multiply. `α(m,i)` is a **ROM read**. The entire apparatus is three adders and a lookup.

**The framing claim of HELICON:** the Black–Scholes–Merton valuation is not a formula to be evaluated. It is a **descent down the curvature ladder** — a trajectory that begins in hyperbolic space, passes through linear space, and terminates in Euclidean space, and the risk sensitivities are not computed at the end of that trajectory. They are the *residue left in the registers along it.*

### 1.1 The Ladder, Mapped

Given spot `S`, strike `K`, rate `r`, carry `q`, volatility `σ`, tenor `T`:

| Rung | Quantity | Geometry | Mode | Seed |
|:--|:--|:--|:--|:--|
| **R1** | `ln(S/K)` | m = −1 | vectoring | x₀ = w+1, y₀ = w−1, w = S/K → z_N = ½·ln w |
| **R1′** | `√T` | m = −1 | vectoring | x₀ = T+¼, y₀ = T−¼ → x_N = K_h·√T |
| **R2** | `σ√T` | m = 0 | rotation | linear multiply |
| **R3** | `d₁, d₂` | m = 0 | vectoring | shift-based division by σ√T |
| **R4** | `e^(−rT)`, `e^(−qT)` | m = −1 | rotation | x₀ = y₀ = 1/K_h → x+y = e^z |
| **R5** | `n(d₁)` | m = −1 → 0 | rotation | e^(−d₁²/2) then scale by 1/√(2π) |
| **R6** | `N(d₁), N(d₂)` | **see §5** | — | Φ is *not* ladder-native |
| **R7** | Price, Δ, Γ, ν, Θ, ρ | m = 0 | rotation | recombination tree |

Note R1 and R1′ share **the same silicon in the same mode**. Note R4 uses that silicon again with the sign of the decision variable flipped. Note R2, R3, R7 are all linear-mode — meaning the multiplier-free machine performs its multiplications and divisions in the *same* structure that performed its logarithms.

The B-58's navigator and the option book share a datapath. That is the whole idea.

---

## 2 · Law I — Residue Harvesting

**The full Greek set costs almost nothing beyond the price.**

Expand the standard sensitivities and something structural appears. Every one of them is an algebraic recombination of exactly **five primitives**:

```
P1 = N(d₁)      P2 = N(d₂)      P3 = n(d₁)
P4 = e^(−rT)    P5 = σ√T                     (with e^(−qT) as P4′)
```

| Greek | In terms of primitives | Marginal cost over price |
|:--|:--|:--|
| Price `C` | `S·P4′·P1 − K·P4·P2` | — |
| **Δ** | `P4′·P1` | **0 ops** (already in a register) |
| **Γ** | `P4′·P3 / (S·P5)` | 1 linear divide |
| **ν** (Vega) | `S·P4′·P3·√T` | 1 linear multiply |
| **Θ** | `−S·P4′·P3·σ/(2√T) − rK·P4·P2 + qS·P4′·P1` | 2 mult, 2 add |
| **ρ** | `K·T·P4·P2` | 1 linear multiply |
| **Vanna** | `−P4′·P3·d₂/σ` | 1 linear multiply |
| **Volga** | `S·P4′·P3·√T·d₁d₂/σ` | 2 linear multiplies |
| **Charm** | `P4′·[qP1 − P3(2(r−q)T − d₂P5)/(2T·P5)]` | 3 mult, 2 add |

The conventional pipeline computes a price and then *re-enters* the machine to differentiate it. HELICON never re-enters. **Residue harvesting** taps live pipeline registers at the stage where each primitive is already resident, routes them into a shallow linear-mode recombination tree, and emits the full nine-Greek vector in the same cycle as the price.

### 2.1 The Cancellation That Makes Δ Free

Delta is `N(d₁)` — a bare probability, not a derivative expression — because of an exact identity:

```
S · e^(−qT) · n(d₁)  ≡  K · e^(−rT) · n(d₂)
```

Worked at S = K = 100, r = 5%, q = 0, σ = 20%, T = 1:

```
S·e^(−qT)·n(d₁) = 37.5240346917
K·e^(−rT)·n(d₂) = 37.5240346917
difference      =  0.000000e+00
```

Zero. Not small — *identically* zero. The two density terms in ∂C/∂S annihilate, which is why Δ collapses to a single CDF evaluation. This identity is not a convenience. In §3 it becomes the single most consequential fact about where you are allowed to spend bits.

### 2.2 Vega and Gamma Are the Same Number

```
ν / Γ  =  S² · σ · T        (exactly)

At S=100, σ=0.20, T=1:   37.5240346917 / 0.0187620173  =  2000.0
                          S²σT                          =  2000.0
```

Two "different" risk numbers that risk systems compute independently are a fixed scalar apart. HELICON computes one and shifts. A Vega/Gamma consistency violation on-chip is therefore not a rounding artifact — it is a **fault signal**. See §6.

---

## 3 · Law II — The Sensitivity Budget

**Bit-width is not a global constant. It is a per-module allocation derived from the Greek Jacobian.**

This is the principle that separates HELICON from every uniform-precision fixed-point pipeline, and it falls directly out of §2.1.

Because the density terms cancel identically in ∂C/∂S, an error **ε** in the hyperbolic-vectoring output `ln(S/K)` has **no first-order effect on the price**. The leading term is quadratic:

| ε in ln(S/K) | Δ price | ε⁻² · Δprice |
|:--|:--|:--|
| 1e−03 | −9.373136e−05 | −93.73 |
| 1e−04 | −9.380226e−07 | −93.80 |
| 1e−05 | −9.380933e−09 | −93.81 |
| 1e−06 | −9.379875e−11 | −93.80 |

The ratio pins at ≈ **−93.8** across four decades. Price error is **O(ε²)**, coefficient ≈ 93.8 for this parameterization.

Now the same ε against Gamma:

| ε in ln(S/K) | rel. Δ Gamma | ε⁻¹ · relΔ |
|:--|:--|:--|
| 1e−03 | −1.760948e−03 | −1.7609 |
| 1e−04 | −1.751097e−04 | −1.7511 |
| 1e−05 | −1.750110e−05 | −1.7501 |
| 1e−06 | −1.750011e−06 | −1.7500 |

Pinned at **−1.7500**, and that constant is not empirical. It is `−d₁/(σ√T) = −0.35/0.20 = −1.75`, exactly, because `∂ln n(d₁)/∂d₁ = −d₁` and `∂d₁/∂ε = 1/(σ√T)`.

### 3.1 The Allocation Rule

```
bits(module_j)  ≥  log₂( |∂G_max/∂m_j| / τ_G )  +  guard(N_j)
```

where `G_max` is the most sensitive output the module feeds, `τ_G` is that output's error tolerance, and `guard` is the CORDIC accumulation allowance (§8.2). The consequences are concrete and counterintuitive:

- **The logarithm stage may run short for a pricer.** To hit 1e−9 in price you need ε ≲ √(1e−9/93.8) ≈ 3.3e−6 → about **19 fractional bits**.
- **The same stage must run long for a risk engine.** To hit 1e−9 *relative* in Gamma you need ε ≲ 5.7e−10 → about **31 fractional bits**. A twelve-bit gap between two modes of the same silicon.
- **The gap widens with moneyness and shrinks with tenor**, since the coefficient is `d₁/(σ√T)`. Deep-OTM short-dated contracts — precisely the ones that dominate modern volume — are where the logarithm stage needs the most bits, and where a uniform-width design silently wastes them everywhere else.

A uniform Q*.40 datapath spends the *same* forty fractional bits on a rung where nineteen would do and a rung where thirty-one are mandatory. HELICON spends them where the Jacobian says they buy something.

---

## 4 · Law III — The Gaussian Gate

**Φ is the only quantity in Black–Scholes that the curvature ladder cannot reach.**

`ln`, `√`, `exp`, `sinh`, `cosh`, `arctan`, multiply, divide — all native. The cumulative normal is not an elementary function in the CORDIC sense. It has no rotation that produces it. This is the architecture's true critical path and every honest design must confront it directly.

### 4.1 The Surrogate Nobody Connected

There exists a closed form that approximates Φ using **only tanh** — a function the hyperbolic engine already emits for free:

```
Φ(x) ≈ ½ · ( 1 + tanh( √(2/π) · ( x + 0.044715·x³ ) ) )
```

This expression is not from quantitative finance. It is the tanh form of the Gaussian error linear unit, published in 2016 as a neural-network activation and shipped inside BERT, GPT-2 and the transformer lineage ever since. `√(2/π) = 0.7978845608`.

Deep learning needed a cheap Gaussian CDF. Option pricing needed a cheap Gaussian CDF. They solved it eight years and one discipline apart, and the machine that computes hyperbolic rotations natively is the machine that gets the deep-learning answer for free.

### 4.2 What It Actually Costs

Measured across x ∈ [−10, 10] at 4·10⁶ points against a double-precision reference:

| Form | Max |ε| | Location | Ladder-native? |
|:--|:--|:--|:--|
| **tanh / GELU form** | **1.7893e−04** | x = 2.5921 | **Yes** — hyperbolic rotation |
| A&S 26.2.17 (Zelen–Severo) | **7.4517e−08** | x = 0.7173 | No — needs `exp` + 5-term Horner |
| Pólya `½(1+√(1−e^(−2x²/π)))` | 3.1458e−03 | — | Partly |
| Logistic `1/(1+e^(−1.702x))` | 9.4863e−03 | — | Yes |

The A&S measurement lands at 7.4517e−08 against its published bound of 7.5e−08 — the classical result reproduces to the fourth digit.

### 4.3 The Two-Tier Gate

Neither form alone is correct for a risk engine. HELICON therefore splits Φ by tier, because **different Greeks need different Φ**:

```
TIER-A  (Δ, ρ, price-in-vol-space, quote generation)
        tanh surrogate, 1.8e−4 absolute
        cost: 1 hyperbolic rotation + 1 cube (2 linear) + 1 shift
        depth: N_h stages, fully shared with the exp module

TIER-B  (price marking, margin, P&L attribution, settlement)
        A&S 26.2.17 with t = 1/(1 + 0.2316419·|x|)
        b = [ 0.319381530, −0.356563782, 1.781477937,
             −1.821255978,  1.330274429 ]
        Φ(x) = 1 − φ(x)·Σ bᵢtⁱ  for x ≥ 0; mirror below
        cost: 1 hyperbolic (e^(−x²/2)) + 1 linear divide + 5 linear mult
        depth: N_h + 8 stages
```

Tier-A is the hot path. Tier-B is the mark. The `φ(x)` factor Tier-B needs is `n(d₁)` — **already computed at rung R5**. Residue harvesting again: Tier-B's expensive term is free because Tier-A's pipeline produced it.

The prediction that follows in §14 is that Tier-A wins the market, because 1.8e−4 in Φ maps to sub-tick error on a $100 underlying, and a quoting engine is not a settlement engine.

---

## 5 · Law IV — The Residual Monitor

**The pricing PDE is a conservation law. Wire it as a checksum.**

Any correct set of outputs must satisfy the Black–Scholes–Merton partial differential equation identically:

```
Θ  +  (r − q)·S·Δ  +  ½·σ²·S²·Γ  −  r·C   ≡   0
```

Measured on the reference point:

```
Θ  = −6.4140275464
Δ  =  0.6368306512
Γ  =  0.0187620173
C  = 10.4505835722

residual = 2.220446049250313e−16      ← one double-precision ULP
```

That residual is a **hardware invariant available at zero marginal latency.** The recombination tree already holds Θ, Δ, Γ and C in adjacent registers in the same cycle. Computing the residual costs three linear-mode multiplies and three adds, running *beside* the output stage rather than in front of it.

```
                 ┌──────────────┐
   R7 outputs ──►│ recomb tree  │──► Δ Γ ν Θ ρ C  (cycle n)
                 └──────┬───────┘
                        │ tap
                 ┌──────▼───────┐
                 │ PDE residual │──► |ε| > τ  ⇒  FAULT flag (cycle n)
                 └──────────────┘
```

What it catches, with no golden reference and no host round-trip:

- a stuck bit anywhere in the hyperbolic ROM
- a CORDIC stage whose repeat index was dropped (§7.2)
- a pre-scaling shift that pushed an input outside the convergence window
- an SEU in a pipeline register
- a scale-factor correction applied the wrong number of times
- silent overflow in the integer field

Add the §2.2 identity as a second, cheaper invariant:

```
|ν − Γ·S²σT|  >  τ₂   ⇒   FAULT
```

Two conservation laws, four adders, one cycle. A pipeline that self-attests is a different risk object than a pipeline that merely runs fast — and in a domain where a wrong Delta on a large book is a solvency event rather than a latency event, this is the feature that matters most and gets designed last.

---

## 6 · Law V — The Bachelier Bypass

**A machine hard-wired to `ln(S/K)` cannot price a market that has gone negative. This is not hypothetical.**

On **20 April 2020**, the May NYMEX WTI contract traded to an intraday low of **−$40.32** and settled at **−$37.63** per barrel. The spot at Cushing reached −$36.58.

Twelve days before that, on 8 April, CME Clearing had published Advisory 20-152, warning members that the clearinghouse would give one day's notice before permitting negative option pricing and strikes, so that firms could switch models. On 21 April, Advisory 20-171 arrived under the subject line **"Switch to Bachelier Options Pricing Model."** It took effect for the margin cycle at end of trading on **22 April 2020**. ICE followed. Within days the market was trading a zero strike on the June future, then −20, then −50.

The reason is stated plainly in the regulatory record: models of the Black–Merton–Scholes type rely on logarithms, and a logarithm has no value at a negative argument.

Now read that as a hardware statement. **A hyperbolic vectoring stage seeded with `x₀ = w+1, y₀ = w−1` has no convergence path for w ≤ 0.** It does not produce a wrong answer. It produces an undefined one, and the pipeline downstream of it produces undefined Greeks at 400 million per second with perfect determinism.

### 6.1 The Bypass Is Free

The 1900 model that Black–Scholes displaced turns out to be a **strict subset** of the same hardware:

```
BLACK–SCHOLES (log-space)          BACHELIER (price-space)
  w  = S/K                           Δ  = F − K
  R1 hyperbolic ln(w)      ✗         (skip — no logarithm exists)
  R1′ hyperbolic √T        ✓         R1′ hyperbolic √T        ✓
  R3 linear divide          ✓         R3 linear divide          ✓
  R6 Φ gate                 ✓         R6 Φ gate                 ✓
  C  = S·N(d₁) − Ke^(−rT)N(d₂)      C = e^(−rT)[(F−K)Φ(d) + σ√T·φ(d)]
```

Bachelier needs `√T`, a subtraction, a division, `Φ` and `φ`. HELICON has all five. It needs **no** logarithm.

So the bypass is a multiplexer on rung R1 and a different recombination constant set on R7. Roughly 2% of the area of the hyperbolic module — and it is the difference between an engine that goes dark on 20 April 2020 and one that does not.

```
              ┌─────────────────────────────┐
  S, K ──────►│ sign(S)·sign(K) guard        │
              └────┬───────────────┬─────────┘
                   │ both > 0      │ either ≤ 0
             ┌─────▼──────┐  ┌─────▼──────────┐
             │ R1: ln(S/K)│  │ BYPASS: F − K  │
             │ hyperbolic │  │ 1 subtract     │
             └─────┬──────┘  └─────┬──────────┘
                   └───────┬───────┘
                     ┌─────▼──────┐
                     │ R3 linear ÷│  → shared tail
                     └────────────┘
```

**Design law:** any engine whose only path to `d₁` runs through a logarithm has embedded an economic assumption — limited liability — into its routing. Assumptions in silicon do not update on a clearing advisory.

---

## 7 · Law VI — Closed Rotation Inversion

**The forward map is not the product. The inverse map is.**

Market makers do not quote prices. They quote volatility, and then convert. The hot path in a live options business runs *backwards*: from an observed premium to the σ that reproduces it. The forward pricer is a subroutine inside that loop, called two to four times per inversion.

And the inverse map is where the arithmetic is genuinely hostile. Newton's step is `σ ← σ + (C_mkt − C)/ν`, so the conditioning is `1/ν`. Vega at one day to expiry, σ = 20%:

| K/S | price | Vega | 1/Vega |
|:--|:--|:--|:--|
| 0.95 | 5.013e+00 | 1.168044e−05 | 8.561e+04 |
| **1.00** | **4.245e−01** | **2.087809e+00** | **4.790e−01** |
| 1.05 | 3.580e−07 | 4.364409e−05 | 2.291e+04 |
| 1.10 | 5.768e−21 | 2.468353e−18 | 4.051e+17 |
| 1.20 | 2.556e−69 | 3.908082e−66 | 2.559e+65 |
| 2.00 | 0.000e+00 | 0.000e+00 | ∞ |

Move 10% out of the money on a one-day option and the Newton denominator has fallen **eighteen orders of magnitude**. In Q24.40, where the ULP is 9.095e−13, Vega at K/S = 1.10 is **six orders of magnitude below one ULP**. It is not small. It is *absent from the number system.*

Any naive fixed-point inversion loop divides by zero in the wings, and the wings are where 0DTE volume lives.

### 7.1 The HELICON Inversion Structure

The published resolution in software is to abandon raw price-space Newton entirely: branch on log-moneyness into rational initial guesses, apply nonlinear transformations to the input price in the branches where it degenerates, and iterate with a fourth-order Householder step. That construction — set out in *Let's Be Rational* (Wilmott, 2015, following *By Implication*, 2006) — reaches maximum attainable double precision in as few as two iterations across all admissible inputs, and is the de facto production standard. Recent published timings put it near **180 ns per evaluation** on a modern core.

HELICON maps that structure onto the ladder rather than re-deriving it:

```
STAGE I   Branch select on x = ln(F/K)   ← R1 output, already resident
          4 rational branches, ROM coefficients, 1 linear Horner tree
          depth: 6 stages

STAGE II  Transform in degenerate branches
          normalized Black coordinates; total-variance parameterization
          w = v² avoids one square root in the loop
          depth: 4 stages

STAGE III Householder(3), order-4 convergence
          numerator/denominator both rational in the residual
          ν, Vanna, Volga are ALREADY HARVESTED (§2) — the second
          and third derivatives the Householder step needs cost
          ZERO additional CORDIC depth
          depth: 2 × (forward pipeline) + 12
```

**This is the payoff of residue harvesting.** A Householder step of order four needs `∂C/∂σ`, `∂²C/∂σ²`, `∂³C/∂σ³`. The first two are Vega and Volga, which fall out of the recombination tree for two multiplies. Conventional implementations pay for them. HELICON already has them in a register — meaning the highest-order, fastest-converging inversion step is the *cheapest* one to build here, exactly inverting the usual cost ordering.

The engine's throughput claim should therefore be stated in **inversions per second**, not prices per second. That is the unit the business buys.

---

## 8 · The Convergence Atlas

Every number below is a hard boundary. Crossing one does not degrade accuracy; it produces garbage with full confidence.

### 8.1 Exact Constants

| Quantity | Circular (m=+1) | Hyperbolic (m=−1) |
|:--|:--|:--|
| Σ micro-angles θ_max | **1.743286620 rad** (99.8830°) | **1.118173016 rad** (64.0664°) |
| tanh(θ_max) | — | **0.806932494** |
| Scale factor K | **1.646760258** | **0.828159361** |
| 1/K | **0.607252935** | **1.207497068** |

θ_max converges to nine digits by N = 30 and does not move thereafter. The scale factors are stable to nine digits from N = 20.

### 8.2 The ln Convergence Window

Using `ln(w) = 2·artanh((w−1)/(w+1))`, the admissible domain is set by `|(w−1)/(w+1)| ≤ tanh(θ_max)`:

```
w_lo = (1 − 0.806932494)/(1 + 0.806932494) = 0.106848
w_hi = (1 + 0.806932494)/(1 − 0.806932494) = 9.359071

              ADMISSIBLE:  w ∈ [ 0.106848 , 9.359071 ]
```

This window is **narrower than it looks**, and the failure mode is silent. A pre-scaler that targets `[0.1, 9.5]` places both endpoints *outside* convergence. The correct barrel-shift target is `w ∈ [0.125, 8.0]` — powers of two safely interior, with the shift count added back as `k·ln2` in the linear tree:

```
S/K = 2^k · w′,  w′ ∈ [0.125, 8.0]
ln(S/K) = k·ln2 + ln(w′)         ln2 = 0.693147180559945
```

`k` comes from a leading-one detector on the ratio — combinational, one stage, no iteration.

### 8.3 The Repeat Schedule

Hyperbolic mode does not converge on the plain index sequence. Iterations **4, 13, 40, 121, …** must each execute **twice**, following `i_{k+1} = 3·i_k + 1`.

This has a physical consequence that is easy to get wrong in a stage count:

| Nominal N | **Physical stages** |
|:--|:--|
| 12 | 13 |
| 16 | 18 |
| 24 | 26 |
| 32 | 34 |
| **40** | **43** |
| 48 | 51 |

**N = 40 is 43 stages, not 40.** The repeats at 4, 13 and 40 add three. A pipeline sized on the nominal count is three stages short of convergence and its scale factor is wrong, because K_h is a product over the *executed* sequence including duplicates. This is the single most common structural fault in hyperbolic implementations, and §5's residual monitor catches it in one cycle.

Also: hyperbolic mode has **no i = 0 iteration** — `artanh(2⁰)` is unbounded. Indexing starts at 1.

---

## 9 · Fixed-Point Engineering: Q24.40

### 9.1 The Format

```
┌─ S ─┬──── 23 integer bits ────┬────────── 40 fractional bits ──────────┐
│  1  │        2^22 … 2^0       │        2^−1 … 2^−40                    │
└─────┴─────────────────────────┴────────────────────────────────────────┘
 64-bit total

 Range      : ± 8,388,608
 Resolution : 2^−40 = 9.094947e−13
```

The integer field is chosen for headroom, not for prices. A $700,000 share price occupies 20 bits and leaves three; the field exists to absorb **exponential excursion** in the R4 rung, where `e^z` for moderately large `z` can exceed the price magnitude by orders. A tenth-of-a-basis-point tick on a $700,000 quote is 1.0e−04, which is **109,951,162 ULPs** — the fractional field is nowhere near the binding constraint on price representation. It is binding on *Vega in the wings* (§7), which is the opposite of where uniform-precision intuition points.

### 9.2 Guard Bits Are Not Optional

CORDIC error has two components: the truncation of the micro-angle table and the accumulation of rounding across stages. The accumulation term grows with stage count, and the classical result puts the requirement at roughly `⌈log₂ N⌉` guard bits plus one for the angle approximation:

| N | Guard bits | Min. fractional width for N clean bits |
|:--|:--|:--|
| 16 | 5 | 21 |
| 24 | 6 | 30 |
| 32 | 6 | 38 |
| **40** | **7** | **47** |
| 48 | 7 | 55 |

**Consequence for Q24.40 at N = 40:** forty fractional bits carrying seven bits of accumulated noise deliver about **33 clean fractional bits ≈ 1.2e−10**. That comfortably clears a 1e−6 target — but the reasoning matters. It clears it *with margin computed*, not *by coincidence of matching numerals*. A format whose fractional width happens to equal its iteration count is a naming collision, not a design.

To hold 40 genuinely clean fractional bits, the datapath wants **Q24.47** — 71 bits internal, truncated to 64 at the output boundary.

### 9.3 Scale-Factor Correction

Hyperbolic vectoring emits `K_h · f(x)`, not `f(x)`. Three options:

1. **Constant multiply by 1/K_h = 1.207497068** — one linear-mode rotation, 1 stage, exact to format.
2. **Fold into the recombination constants** — free, but only when the result feeds a multiply anyway (true for R1′ → R2, false for R1 → R3).
3. **Repeated-stage compensation** — inserting extra shift-add stages that drive the product toward 1. Zero multipliers, but it perturbs the repeat schedule in §8.3 and must be re-derived, not copied.

Option 2 wherever it applies; option 1 elsewhere. Option 3 is a trap on a design that already has a fixed repeat sequence.

---

## 10 · The Latency–Precision Frontier

This is the table that decides whether a design is real. Radix-2, 400 MHz, 2.5 ns per stage. Critical path modelled as three serial ladder modules (`ln` ∥ `√T` → divide → `exp` ∥ `Φ`) plus a 20-stage recombination and monitor tail.

| N | ULP ≈ 2⁻ᴺ | Stages/module | Module latency | **Critical path** |
|:--|:--|:--|:--|:--|
| 12 | 2.441e−04 | 13 | 32.5 ns | **147.5 ns** |
| 14 | 6.104e−05 | 16 | 40.0 ns | **170.0 ns** |
| 16 | 1.526e−05 | 18 | 45.0 ns | **185.0 ns** |
| 20 | 9.537e−07 | 22 | 55.0 ns | **215.0 ns** |
| 24 | 5.960e−08 | 26 | 65.0 ns | **245.0 ns** |
| 28 | 3.725e−09 | 30 | 75.0 ns | **275.0 ns** |
| 32 | 2.328e−10 | 34 | 85.0 ns | **305.0 ns** |
| **40** | **9.095e−13** | **43** | **107.5 ns** | **372.5 ns** |
| 48 | 3.553e−15 | 51 | 127.5 ns | **432.5 ns** |

**Read the frontier honestly.** At 400 MHz on a radix-2 ladder:

- **A 115 ns budget buys 46 stages total.** Split three ways that is ~15 stages per module — nominal N ≈ 12–13, ULP ≈ 2.4e−4. That is a *quoting* engine, and a good one.
- **A 1e−6 target needs N ≥ 20**, i.e. ≥ 215 ns. Marking accuracy and 115 ns are not simultaneously available in this technology at this radix.
- **Full N = 40** — the depth that justifies forty fractional bits — is **372.5 ns**, and `ln` alone is 107.5 ns of it.

### 10.1 Buying the Frontier Back

| Lever | Effect | Cost |
|:--|:--|:--|
| **Radix-4** | ~N/2 stages: N=40 → **23 stg / 57.5 ns**; N=32 → 19 stg / 47.5 ns | Wider adders, redundant sign selection, lower f_max |
| **Parallel issue R1 ∥ R1′** | −43 stages on the path | 2× hyperbolic area (already assumed above) |
| **Tier-A Φ** (§4.3) | −8 stages | 1.8e−4 in Φ |
| **Speculative branch on d₁ sign** | −2 stages | 2× recombination tail |
| **Total-variance `w = v²` in the inverse loop** | −1 √ per iteration | none |

Radix-4 on the two hyperbolic modules plus parallel issue plus Tier-A Φ lands **N=40 accuracy in ≈ 200 ns**, or **N=24 accuracy in ≈ 125 ns**. Those are the two defensible operating points. Anything claiming full double-matching accuracy at ~115 ns on a 400 MHz radix-2 ladder has counted 40 stages where the repeat schedule requires 43, and has not put `Φ` on the critical path at all.

### 10.2 Throughput Is Independent of All of This

Deep pipelining decouples latency from rate. **One complete Greek set per clock**, every clock, regardless of depth:

```
400 MHz × 1 set/cycle           = 4.00e8 Greek-sets/s per lane
4 lanes                          = 1.60e9 Greek-sets/s
Single core, LBR at 180 ns/eval  = 5.56e6 inversions/s

Ratio, one lane : one core       ≈ 72 ×
```

And the number that gives that scale meaning: **OPRA peak throughput reached 50.9 million messages per second in January 2026 and 53.5 million in February 2026.** During the April 2025 sell-off, one-millisecond bursts exceeded 23.7 million packets per second — over 187 million messages per second.

```
One HELICON lane / peak OPRA rate  =  4.00e8 / 5.35e7  ≈  7.5 ×
Cores needed to match one lane      ≈  72
Cores needed to invert every OPRA
message at peak                     ≈  10
```

A single lane re-prices the entire consolidated US options tape, at its record peak rate, seven times over, with a full nine-Greek vector per message and a PDE residual on each one. The engine is not throughput-bound by the market. It is bound by how many *scenarios* you want per message — and that is the actual design question, which the throughput framing has been obscuring for a decade.

---

## 11 · Resource Physics: The Crossover Nobody Publishes

The founding premise — "eliminate multipliers" — was formulated for a machine that had none. Modern reconfigurable fabric is not that machine.

### 11.1 One Ladder Stage, Costed

```
per stage (64-bit):  2 × add/sub (x,y paths)  +  1 × add/sub (z path)
                     shifts are ROUTING, not logic — genuinely free
64-bit add on UltraScale+: 8 × CARRY8, ~64 LUT + 64 FF
per stage:  ~192 LUT + ~192 FF
N=40 module (43 stages):  ~8,256 LUT + ~8,256 FF,  ZERO DSP
```

### 11.2 The Fabric It Runs On

| Device | LUTs | Registers | DSP | On-chip mem |
|:--|:--|:--|:--|:--|
| **Alveo U280** (UltraScale+, 16 nm) | 1,304 K | 2,607 K | **9,024** DSP48E2 | 2,016 BRAM + 960 URAM, 8 GB HBM2 @ 460 GB/s |
| **Alveo U250** (UltraScale+, 16 nm) | ~1,300 K usable | — | ~11,500 usable | 64 GB DDR4 |
| **Versal VC1902** (7 nm, 37 B transistors) | ~900 K | — | DSP58 (27×24, native FP) | 855 Mb; **400 AI Engine tiles**, 50×8 |

DSP48E2 is a 27×18 multiplier. **DSP58 is a strict superset — 27×24, 58-bit logic unit, and native floating point.** That last capability is the one that changes the argument.

### 11.3 The Honest Comparison

| Path | Latency | LUTs | DSPs |
|:--|:--|:--|:--|
| CORDIC `ln`, N=40 | 43 cycles | **~8,256** | **0** |
| Minimax poly `ln`, deg-9 Horner on DSP58 | ~45 cycles | **~0** | **9** |

**They are the same latency.** The trade is not speed. It is **LUTs against DSPs** — a placement decision, not an algorithmic one.

On a U280 the logic budget supports roughly **157 concurrent N=40 hyperbolic modules** on LUTs alone, while 9,024 DSP48E2 supports on the order of 900 concurrent double-precision multiplies. Neither resource is scarce in isolation. What is scarce is *both at once*, in a design that also has to fit a 100G MAC, a feed handler, an order book, and a risk aggregator on the same die.

**The correct rule:** CORDIC is not faster. CORDIC is **DSP-free**, and DSP-free is worth exactly as much as your DSP columns are contended. On a part where the transcendental path competes with an inference accelerator for DSP58s, CORDIC wins decisively. On a part where DSPs sit idle, a polynomial wins on area and closes timing more easily. The premise "replace multipliers" was a hardware conclusion from 1956 restated as a mathematical principle. It is a **budget statement**, and budgets are per-design.

---

## 12 · The Light Budget

Some numbers to hold the pipeline against.

### 12.1 One Clock Cycle, in Metres

| Clock | Cycle | Light in vacuum | Light in fibre (n = 1.4682) |
|:--|:--|:--|:--|
| 250 MHz | 4.000 ns | 119.92 cm | 81.68 cm |
| **400 MHz** | **2.500 ns** | **74.95 cm** | **51.05 cm** |
| 700 MHz | 1.429 ns | 42.83 cm | 29.17 cm |

Light crosses one foot in **1.0167 ns**. A 372.5 ns HELICON traversal is the time light needs to travel **111.7 metres** in vacuum — roughly the length of a football pitch. A 147.5 ns traversal is 44 metres.

Every stage in §8.3's repeat schedule is 75 cm of light. The three extra stages the hyperbolic repeats demand — the ones a nominal stage count omits — cost **2.25 metres of light** and are the difference between convergence and confident nonsense.

### 12.2 The Transport Is Finished

Aurora, Illinois to Carteret, New Jersey — the CME matching engine to the Nasdaq data centre:

```
Great-circle distance        1,185.5 km
Vacuum round trip            7.909 ms      ← physical floor
Best fibre round trip       11.612 ms
```

In August 2010, a purpose-built dark-fibre route between those two points went live at **13.33 ms** round trip after roughly **$300 million** of construction across 825 miles. Route refinements brought it to **12.98 ms** by October 2012. Microwave then took the same path through air rather than glass and walked it down through 10, 9, 8.5, to about **8.1 ms** — with published operator targets below **8.05 ms** to Secaucus and below **8.00 ms** to Carteret.

```
Best microwave       ≈ 8.00–8.10 ms
Vacuum floor            7.909 ms
REMAINING HEADROOM   ≈ 1.2 – 2.4 %
```

**The transport layer has essentially been solved to the speed of light.** A $300 million fibre asset was second-best within two years and the medium that beat it is now within a couple of percent of a bound that no amount of capital moves.

This is the fact that reframes everything upstream. When the wire is finished, the only remaining variable is what happens **at the endpoint** — which is to say, in the gates. HELICON's 372.5 ns is 0.0047% of an 8 ms round trip. And that is precisely why it is now the interesting number: it is the last term in the budget that is still elastic.

### 12.3 The Counter-Move

Not every venue rewards the last nanosecond. IEX interposes a physical delay line — historically **38 miles** of coiled fibre, extended to about **44 miles** after a 2024 data-centre relocation to hold the same figure — producing a uniform **350 µs** delay on inbound orders. The stated logic is that the referee should be **faster than your fastest participant**.

```
350 µs in fibre  =  71.47 km  =  44.41 miles       ← matches the published coil
HELICON traversal / IEX bump  =  372.5 ns / 350 µs  =  0.106 %
```

On a speed-bumped venue the entire pipeline is one part in a thousand of the mandated delay. Latency stops being the figure of merit. **Sets per second per watt per rack unit** becomes the figure of merit — which is the regime where a deeply pipelined, branchless, jitter-free fixed-point engine is at its strongest, and where a floating-point CPU farm is at its weakest.

---

## 13 · The Reversal

Here is the thing that ought to be uncomfortable, and that the architecture has to answer.

Everything above builds a machine of extraordinary precision to evaluate a formula whose central assumption the market rejected almost forty years ago.

Before 19 October 1987, implied volatility across strikes was close to flat — which is what constant-σ predicts. In the pre-crash record, ten-percent out-of-the-money one-month puts carried an average implied-vol spread of **1.83%** over at-the-money, and 2.5% in-the-money puts averaged **−0.12%**. Small, unsystematic, model-consistent.

On that Monday, the skew appeared. It never went away. Deep out-of-the-money short-dated index puts have been persistently underpriced by the flat-σ formula in every sample since.

So the formula is wrong, and everybody knows it is wrong, and everybody uses it anyway — because implied volatility is, in the best-known description of it, **"the wrong number to put in the wrong formula to get the right price."**

Which raises the obvious objection: why build 43-stage hyperbolic ladders and 7 guard bits and a PDE residual monitor for a model that is admitted to be false?

**Because the formula's job changed, and its new job has stricter requirements than its old one.**

Black–Scholes stopped being a pricing model in 1987 and became a **coordinate system** — a bijection between the price a market quotes and the σ that market speaks in. A coordinate transform does not need to be *true*. It needs to be **exactly invertible, identically implemented everywhere, and deterministic**, because two desks quoting the same contract must map to the same σ or they cannot trade with each other at all.

That is a *harder* specification than accuracy, and it is one that fixed-point hardware satisfies better than floating-point software:

- **Bit-exact reproducibility.** No FMA contraction, no reassociation, no `-ffast-math`, no compiler version. The same inputs give the same bits on every die, forever.
- **Zero jitter.** Branchless execution means the 10,000th contract in a burst takes exactly as long as the first. No cache miss, no branch mispredict, no interrupt, no page fault. Under a 187-million-message burst, a floating-point farm's *tail* latency is the number that kills you, and a systolic pipeline has no tail.
- **Attestable correctness.** §5's residual is a per-result proof, not a nightly regression.

The engine's value was never accuracy against the true price, because there is no true price. Its value is that it is **the same wrong number, everywhere, every time, provably, at 400 million per second.**

That is what the 1959 navigation computer delivered too. The B-58 did not need the true position. It needed the same position, every cycle, on time, without exception — because a bomber that occasionally reports a very good position and occasionally stalls is worse than one that always reports an adequate one.

Sixty-seven years apart, in two fields with no shared literature, the requirement is identical. Determinism outranks accuracy whenever the clock is not negotiable.

---

## 14 · Seven Predictions

Stated so they can be wrong.

**P1 · The transcendental path leaves the fabric by 2029.**
On Versal-class and successor parts, DSP58's native floating-point support plus AI-Engine tiles will make short minimax polynomials beat radix-2 CORDIC on area at equal latency for `ln` and `exp`. CORDIC survives specifically where DSP columns are contended by co-resident inference or filtering logic.
*Falsifier:* a published Versal-generation option-pricing design beating a DSP58 polynomial path on both LUT count and latency, with DSPs otherwise uncommitted.

**P2 · Φ becomes the acknowledged critical path.**
Because every other Black–Scholes primitive is ladder-native and Φ is not, the Gaussian gate will be where designs differentiate. Winning implementations will use a hyperbolic-native surrogate — the tanh form — with a low-order correction, rather than classical rational approximations, because tanh is free on a machine that already rotates hyperbolically.
*Falsifier:* dominant published designs continuing to spend `exp` + 5-term Horner on Φ where a hyperbolic module is already instantiated.

**P3 · Uniform bit-width ends.**
Per-module widths derived from the Greek Jacobian (§3) become standard, driven by the O(ε²)/O(ε) asymmetry between price and Gamma. Expect published designs with 19–22 bits on the logarithm rung for quoting paths and 31+ on the same rung for risk paths, selectable at synthesis.
*Falsifier:* a mature published architecture holding a single global fractional width across all rungs while claiming Gamma accuracy.

**P4 · The product is the inverse map.**
Engines will be specified and sold in **implied-volatility inversions per second**, not prices per second. The forward pricer becomes an unmarketed subroutine. Residue harvesting makes order-4 Householder steps cheaper than order-2 Newton steps on this architecture, inverting the usual cost ordering.
*Falsifier:* datasheets continuing to headline forward-pricing throughput after 2027.

**P5 · Conservation-law monitoring becomes a compliance requirement.**
The PDE residual (§5) and the ν/Γ identity (§2.2) are free per-result attestations. As deterministic hardware moves closer to the margin and settlement path, an auditable per-result invariant becomes something regulators can ask for — because it is the only correctness evidence that exists at line rate.
*Falsifier:* five years of hardware risk deployment with no invariant-monitoring requirement in any venue or clearing rule.

**P6 · Model-agnostic seeding becomes mandatory.**
The April 2020 event was not a tail; it was a demonstration that the log-space assumption is a *listing convention*, not a law. Engines will ship with the Bachelier bypass (§6) as a standard multiplexer, and eventually with displaced-lognormal and normal-SABR seeds sharing the same ladder.
*Falsifier:* a negative-underlying episode after 2026 in which log-only engines suffer no material outage.

**P7 · The figure of merit inverts on speed-bumped venues.**
Where a venue imposes a delay orders of magnitude larger than the pipeline (350 µs versus 372.5 ns — a factor of ~940), published benchmarks will shift from nanoseconds of latency to **Greek-sets per second per watt per rack unit**. Latency-optimised and throughput-optimised builds will diverge into distinct product lines.
*Falsifier:* latency remaining the sole headline metric for options risk hardware through 2028.

---

## 15 · Reference Architecture

```
                         ┌────────────────────────────────┐
   feed  ───────────────►│  ingress: S, K, r, q, σ, T      │
   (OPRA / prop / ITCH)  │  sign guard + leading-one det.  │
                         └───────┬────────────────┬───────┘
                                 │                │
                     ┌───────────▼──┐      ┌──────▼─────────┐
                     │ PRE-SCALER   │      │ BACHELIER MUX  │
                     │ S/K → 2^k·w′ │      │ (§6) F−K path  │
                     │ w′∈[.125,8]  │      └──────┬─────────┘
                     └───────┬──────┘             │
        ┌────────────────────┼─────────────────┐  │
        │                    │                 │  │
  ┌─────▼──────┐      ┌──────▼──────┐   ┌──────▼──▼────┐
  │ R1  ln(w′) │      │ R1′  √T     │   │  (bypass)    │
  │ m=−1 vect  │      │ m=−1 vect   │   │              │
  │ 43 stages  │      │ 43 stages   │   └──────┬───────┘
  └─────┬──────┘      └──────┬──────┘          │
        │  + k·ln2           │ × 1/K_h         │
        └──────────┬─────────┴─────────────────┘
                   │
            ┌──────▼───────┐
            │ R2/R3 linear │  σ√T ; d₁ = (·)/σ√T ; d₂ = d₁ − σ√T
            │ 10 stages    │
            └──────┬───────┘
        ┌──────────┼──────────┐
   ┌────▼─────┐ ┌──▼───────┐ ┌▼──────────────┐
   │ R4 exp   │ │ R5 n(d₁) │ │ R6 Φ GATE     │
   │ m=−1 rot │ │ m=−1→0   │ │ A: tanh 1.8e−4│
   │          │ │          │ │ B: A&S 7.5e−8 │
   └────┬─────┘ └──┬───────┘ └────┬──────────┘
        └──────────┼──────────────┘
             ┌─────▼──────────────────┐
             │ R7 RECOMBINATION TREE  │──► C Δ Γ ν Θ ρ Vanna Volga Charm
             │ m=0, 20 stages         │
             └─────┬──────────────────┘
                   │ tap (same cycle)
             ┌─────▼──────────────┐
             │ RESIDUAL MONITOR   │──► PDE residual, ν/Γ identity, FAULT
             └────────────────────┘
                   │
             ┌─────▼──────────────┐
             │ INVERSION LOOP     │──► σ_implied  (§7, Householder-3)
             │ reuses ν, Volga    │
             └────────────────────┘
```

### 15.1 Layout

```
rtl/
├── helicon_top.vhd            top-level, mode/tier straps, fault aggregation
├── ladder/
│   ├── cordic_hyperbolic.vhd  m=−1, parameterised N, REPEAT SCHEDULE IN GENERIC
│   ├── cordic_circular.vhd    m=+1
│   ├── cordic_linear.vhd      m=0, multiply/divide
│   └── repeat_sched.vhd       generates {4,13,40,121,...}; k→3k+1
├── gates/
│   ├── phi_tier_a.vhd         tanh surrogate, √(2/π)=0.7978845608, c=0.044715
│   ├── phi_tier_b.vhd         A&S 26.2.17, p=0.2316419, b[1..5]
│   └── phi_mux.vhd            per-output tier selection
├── prescale/
│   ├── lead_one.vhd           combinational k extraction
│   └── window_guard.vhd       asserts w′ ∈ [0.125, 8.0]; FAULT otherwise
├── bachelier/
│   └── bypass_mux.vhd         §6; sign guard on S, K, F
├── recomb/
│   ├── greek_tree.vhd         nine outputs, linear mode
│   └── residual_monitor.vhd   §5 PDE + ν/Γ invariants
├── invert/
│   ├── branch_select.vhd      4 rational branches on ln(F/K)
│   └── householder3.vhd       order-4; consumes harvested ν, Volga
└── fixed/
    ├── q24_40_pack.vhd        external format
    └── q24_47_pack.vhd        internal width (§9.2)

anchors/
├── ladder_constants.csv       θ_max, K, 1/K per N — nine digits
├── window_edges.csv           0.106848 / 9.359071 and safe interior targets
├── phi_error_profile.csv      both tiers, |x| ≤ 10, 4e6 points
├── jacobian_budget.csv        per-rung bit allocation vs (d₁, σ√T)
└── invariant_traces.csv       PDE residual under injected stage faults

tools/
├── gen_atanh_rom.py           --bits N --repeats auto --width 47
├── frontier.py                latency/precision table (§10) for a target f_max
└── budget.py                  solves §3.1 for a given tolerance vector
```

### 15.2 Build

```bash
# ROM generation — repeat schedule is DERIVED, never hand-listed
python3 tools/gen_atanh_rom.py \
        --mode hyperbolic --bits 40 --width 47 \
        --repeats auto \
        --out rtl/ladder/rom_atanh.vhd
# emits 43 entries for --bits 40, and asserts on any mismatch

python3 tools/gen_atanh_rom.py \
        --mode circular --bits 40 --width 47 \
        --out rtl/ladder/rom_atan.vhd

# frontier for your actual achieved f_max
python3 tools/frontier.py --fmax 400e6 --radix 2 --serial-modules 3 --tail 20

# per-rung widths for a tolerance vector
python3 tools/budget.py --tol-price 1e-9 --tol-gamma-rel 1e-9 \
                        --d1-range -4 4 --sigma-sqrt-t 0.02 0.60

# elaborate
ghdl -a --std=08 rtl/fixed/*.vhd rtl/ladder/*.vhd rtl/gates/*.vhd \
                 rtl/prescale/*.vhd rtl/bachelier/*.vhd rtl/recomb/*.vhd \
                 rtl/invert/*.vhd rtl/helicon_top.vhd
ghdl -e helicon_top
```

---

## 16 · Hazards

| # | Hazard | Symptom | Guard |
|:--|:--|:--|:--|
| **H1** | Repeat schedule dropped | Silent 3-stage shortfall at N=40; wrong K_h | Derive `{k, 3k+1}` in RTL; PDE residual (§5) |
| **H2** | Pre-scale target outside window | Non-convergence at the edges, no error flag | Target [0.125, 8.0], never [0.1, 9.5]; `window_guard` |
| **H3** | Guard bits omitted | 7 bits of noise presented as signal at N=40 | Q24.47 internal (§9.2) |
| **H4** | `i = 0` included in hyperbolic index | Unbounded table entry | Index from 1 |
| **H5** | Scale factor applied twice | Output off by 1.207497068 | Fold at exactly one site per path; ν/Γ invariant catches it |
| **H6** | Negative or zero underlying | Undefined `ln`, deterministic garbage downstream | Bachelier bypass (§6), sign guard at ingress |
| **H7** | Vega below one ULP in the wings | Inversion divides by zero at K/S ≥ 1.10, T = 1d | Branch-select + transformed objective (§7.1); never raw price-Newton |
| **H8** | Uniform width across rungs | Gamma quietly 1.75× more wrong than price | Jacobian budget (§3.1) |
| **H9** | Tier-A Φ used for settlement marks | 1.8e−4 absolute in a marked book | Tier strap per output class (§4.3) |
| **H10** | Integer overflow in exponential excursion | Wrap, not saturate; sign flip | 23 integer bits + saturating adder on R4 |

---

## 17 · Notation

| Symbol | Value / meaning |
|:--|:--|
| `m` | Curvature parameter: +1 circular, 0 linear, −1 hyperbolic |
| `θ_max` (m=−1) | 1.118173016 rad (64.0664°) |
| `θ_max` (m=+1) | 1.743286620 rad (99.8830°) |
| `K_h` | 0.828159361 — hyperbolic scale factor |
| `1/K_h` | 1.207497068 |
| `K_c` | 1.646760258 — circular scale factor |
| `1/K_c` | 0.607252935 |
| `tanh(θ_max)` | 0.806932494 |
| ln window | [0.106848, 9.359071] — target [0.125, 8.0] |
| Repeat set | 4, 13, 40, 121, … `i→3i+1`; hyperbolic only |
| `√(2/π)` | 0.7978845608 |
| GELU-form cubic coeff | 0.044715 |
| A&S `p` | 0.2316419 |
| A&S `b[1..5]` | 0.319381530, −0.356563782, 1.781477937, −1.821255978, 1.330274429 |
| Q24.40 ULP | 9.094947e−13 |
| Q24.40 range | ± 8,388,608 |
| `ln 2` | 0.693147180559945 |

---

## 18 · Lineage

The ideas this rests on, in order of appearance.

- **1624** — Briggs, *Arithmetica Logarithmica*. Logarithms by shift and add, 332 years early.
- **1900** — Bachelier, *Théorie de la spéculation*. Arithmetic Brownian motion; no logarithm required. Dormant for a century; mandatory in a week in 2020.
- **15 June 1956** — Volder, Convair internal report IAR-1.148, "Binary Computation Algorithms for Coordinate Rotation and Function Generation." Aeroelectronics group.
- **1959** — Volder, "The CORDIC Trigonometric Computing Technique," *IRE Transactions on Electronic Computers*.
- **1971** — Walther, "A unified algorithm for elementary functions." Three geometries, one parameter.
- **1964** — Zelen & Severo, probability chapter in the *Handbook of Mathematical Functions*. Formula 26.2.17, error bound 7.5e−8.
- **1972** — Cochran, "Algorithms and Accuracy in the HP-35," *HP Journal*. CORDIC in a pocket.
- **1973** — Black & Scholes, "The Pricing of Options and Corporate Liabilities," *J. Political Economy* 81(3):637–654. Merton, "Theory of Rational Option Pricing," *Bell J. Econ.* 4(1):141–183.
- **1983** — Nave, "Implementation of Transcendental Functions on a Numerics Processor." CORDIC inside the 80x87.
- **1988** — Black, "The Holes in Black–Scholes," *Risk*. The author, on the model.
- **19 October 1987 → 1994** — Rubinstein, "Implied Binomial Trees," *J. Finance* 49:771–818. The skew, documented and permanent.
- **1991** — Hu, Harber & Bass, "Expanding the range of convergence of the CORDIC algorithm," *IEEE Trans. Computers* 40:13–21.
- **1992** — Hu, "The quantization effects of the CORDIC algorithm," *IEEE Trans. Signal Processing* 40:834–844. Where the guard bits come from.
- **2000** — Volder, "The Birth of CORDIC," *J. VLSI Signal Processing* 25:101–105. Walther, "The Story of Unified CORDIC," same volume, 107–112.
- **2006 / 2015** — Jäckel, "By Implication"; "Let's Be Rational," *Wilmott* (2015), 40–53. Two Householder iterations to attainable precision.
- **2016** — Hendrycks & Gimpel, "Gaussian Error Linear Units (GELUs)," arXiv:1606.08415. The tanh Gaussian CDF, arriving from the wrong field.
- **8 / 21 April 2020** — CME Clearing Advisories 20-152 and 20-171. "Switch to Bachelier Options Pricing Model," effective 22 April 2020. ICE followed.
- **2022** — AMD *Versal ACAP DSP Engine Architecture Manual* (AM004). DSP58: 27×24, 58-bit logic unit, native floating point, superset of DSP48E2.

---

*Every constant in this document is either exact, derived from an exact identity, or measured against a double-precision reference at the stated point count. Where a claim depends on a clock rate, the clock rate is stated. Where a claim depends on a device, the device is named.*
