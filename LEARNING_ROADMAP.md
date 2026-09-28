# FinancePy — Personal Learning & Contribution Roadmap

> **This is a personal file.** It lives on your fork's `master` so you can tick boxes over time.
> Never include it in a Pull Request to `domokane/FinancePy` — keep PR branches created from
> `upstream/master` clean. (See §8.2 for the exact branch workflow.)

**Written for:** 2nd-year data science student, 3 years of investing, knows *concepts* (index
funds, bonds, options) but wants the *math*. Background: Calculus 1–2, Probability 1–2, Linear
Algebra (some rust), Python up to OOP. No stochastic calculus yet.

**Your goal (as stated):** learn the quantitative finance and the mathematics deeply.
Contributing is secondary — it is the *practice ground*, not the objective.

---

## 0. How to use this file

1. Read §1–§4 once, do the environment setup in §4, then **stop reading and start Stage 1 of §6.**
2. Each stage has the same shape:
   - **Goal** — what you should be able to do.
   - **Math to refresh** — the prerequisite you already met once, restated.
   - **Read (in order)** — concrete files, smallest first.
   - **Run** — a notebook or script that already exists.
   - **Your exercise / acceptance test** — a small script *you* write that proves understanding.
3. Do not move on until the acceptance test passes. That habit is the whole method.
4. Stages are ordered **least → most mathematically complex**, which is what you asked for.
5. §7 is the Python skill list. §8–§9 are the contribution track (lower priority, but cheap wins
   are marked).

---

## 1. What this repository is

FinancePy is a **pure-Python derivatives pricing and risk library** by Dominic O'Kane (ex-EDHEC
professor, 12 years industry). It prices and risk-manages: equity / FX / interest-rate / credit
derivatives plus bonds.

Three things make it a genuinely good *learning* repository:

1. **It is written to be read.** From the README: the code is deliberately simple, no Cython, no
   clever one-liners; a loop is preferred over a list comprehension. You can follow logic from the
   public API down to the lowest-level formula.
2. **Speed comes from Numba**, not from obfuscation. Functions are decorated `@njit` / `@vectorize`
   so Python is ~10–100× faster while staying readable. You get to see numerical code written the
   way a quant would write it, but in Python.
3. **It uses a uniform architecture**, so once you learn *one* product you have learned the shape
   of all ~90 of them:

   > **VALUATION = PRODUCT + MODEL + MARKET**
   >
   > `value = product.value(value_date, market_data, model)`

   `Product` = *what* is being priced. `Model` = the *mathematics* of the randomness. `Market` =
   the *observable data* (curves, vols, prices). That separation is the single most important idea
   to internalise, and it is what makes this repo teachable.

Size: 215 Python files, ~76,000 lines in `financepy/` (58 of them in `models/`), 129 example
notebooks, 124 `example_*.py` scripts, and two independent test suites: `tests/unit` (106 pytest
files) and `tests/regression` (123 golden-file test files).

---

## 2. The architecture, concretely

```
VALUATION  =  PRODUCT   +   MODEL   +   MARKET
              ───────       ─────     ────────
              what pays     how it    inputs
              what, when    moves     (curves, vols)

+ UTILS  — dates, day counts, schedules, calendars, solvers: the plumbing
           (and the layer that will confuse you most if you skip it)
```

Read this real example top-to-bottom — it is the canonical path through the library:

```python
from financepy.utils.date import Date
from financepy.utils.global_types import OptionTypes
from financepy.market.curves.flat_discount_curve import FlatDiscountCurve   # MARKET
from financepy.models.black_scholes import BlackScholes                     # MODEL
from financepy.products.equity.equity_vanilla_option import EquityVanillaOption  # PRODUCT

value_dt  = Date(1, 1, 2024)
expiry_dt = Date(1, 1, 2025)

call = EquityVanillaOption(expiry_dt, 100.0, OptionTypes.EUROPEAN_CALL)  # PRODUCT
r = FlatDiscountCurve(value_dt, 0.05)   # MARKET: discount curve
q = FlatDiscountCurve(value_dt, 0.01)   # MARKET: dividend curve
model = BlackScholes(0.20)              # MODEL: 20% volatility

print(call.value(value_dt, 100.0, r, q, model))   # -> 9.84198085425277
print(call.delta(value_dt, 100.0, r, q, model))   # -> 0.6119013320777545
print(call.vega (value_dt, 100.0, r, q, model))   # -> 37.80528689382452
```

*(These three numbers were run and verified in this environment. You will derive them by hand in
Stage 4.)*

---

## 3. Repository map (verified paths)

```
FinancePy/
├── financepy/
│   ├── utils/          <- START HERE. Dates, day counts, schedules, math, solvers
│   │   ├── date.py            1053 lines  Date class (day arithmetic, IMM/CDS dates)
│   │   ├── calendar.py        3099 lines  holiday calendars (biggest file; use, don't read whole)
│   │   ├── day_count.py        301 lines  year-fraction conventions (ACT/360, 30/360, ACT/ACT...)
│   │   ├── schedule.py         275 lines  cashflow date generation
│   │   ├── day_count, compounding, frequency, tenor, amount  <- small, easy reads
│   │   ├── math.py             755 lines  normcdf/normpdf/norminvcdf, Cholesky, tridiagonal solve
│   │   ├── solver_1d.py        852 lines  newton, newton_secant, bisection, brent_max
│   │   ├── solver_nm.py / solver_cg.py    Nelder-Mead / conjugate gradient optimisers
│   │   └── polyfit.py, tension_spline.py  curve fitting
│   ├── market/         <- MARKET DATA
│   │   ├── curves/     flat_discount_curve.py, discount_curve.py, ibor_single_curve.py,
│   │   │               ois_curve.py, cds_curve.py, ns/nss/poly/pwf curve families
│   │   ├── volatility/ equity_vol_curve, equity_vol_surface, fx_vol_surface,
│   │   │               swaption_vol_surface, ibor_cap_vol_curve
│   │   └── prices/     (placeholder)
│   ├── models/         <- THE MATHEMATICS (58 files)
│   │   ├── black_scholes_analytic.py  1158 lines  BS formula + all Greeks + implied vol
│   │   ├── black_scholes.py            329 lines  dispatcher across methods
│   │   ├── black_scholes_mc.py                    Monte Carlo, 6 implementations
│   │   ├── equity_crr_tree.py                     Cox-Ross-Rubinstein binomial tree
│   │   ├── equity_lsmc.py                         Longstaff-Schwartz (regression!) 
│   │   ├── finite_difference.py / finite_difference_psor.py   PDE solvers
│   │   ├── heston.py, sabr.py, sabr_shifted.py, merton_jump_diffusion.py, cev.py, dupire.py
│   │   ├── svi.py / svi_surface.py / ssvi_surface.py          vol surface models
│   │   ├── vasicek_mc.py, cir_montecarlo.py, rates_ho_lee.py  short-rate models
│   │   ├── bdt_tree.py, bk_tree.py, hw_tree.py                trinomial rate trees
│   │   ├── lmm_mc.py                                          LIBOR Market Model
│   │   ├── gauss_copula*.py, student_t_copula.py, loss_dbn_builder.py
│   │   ├── merton_firm.py, merton_firm_mkt.py                 structural credit
│   │   ├── gbm_process_simulator.py, process_simulator.py, sobol.py
│   │   └── model.py                                           abstract base `Model`
│   └── products/       <- THE CONTRACTS (grouped by asset class)
│       ├── bonds/    bond, bond_zero, bond_frn, bond_convertible, bond_option,
│       │             bond_inflation, bond_mortgage, bond_future, bond_portfolio, cashflow
│       ├── credit/   cds, cds_curve, cds_basket, cds_index_option, cds_option,
│       │             cds_index_portfolio, cds_tranche
│       ├── equity/   equity_vanilla_option, equity_american_option, equity_barrier_option,
│       │             equity_asian_option, equity_digital_option, equity_lookback (fixed/float),
│       │             equity_chooser_option, equity_compound_option, equity_cliquet_option,
│       │             equity_rainbow_option, equity_basket_option, equity_variance_swap,
│       │             equity_one_touch_option, equity_swap, equity_forward, ...
│       ├── fx/       fx_vanilla_option, fx_barrier_option, fx_digital_option,
│       │             fx_one_touch_option, fx_double_one_touch_option, fx_forward, ...
│       └── rates/    ibor_deposit, ibor_fra, ibor_future, ibor_swap, ois, ois_basis_swap,
│                     ibor_cap_floor, ibor_swaption, ibor_bermudan_swaption, callable_swap,
│                     inflation_swap, dual_curve, swap_fixed_leg, swap_float_leg
├── examples/
│   ├── notebooks/    129 notebooks, organised utils/ market/ models/ products/
│   └── scripts/      124 `example_*.py` files (same organisation) + run_all_scripts.py
├── tests/
│   ├── unit/         112 pytest files — deterministic edge cases only
│   └── regression/   golden-file comparison framework (FinTestCases.py), not pytest
├── docs/             generated HTML API reference (no .md sources — see §9)
├── mkdocs.yml, pyproject.toml, requirements.txt, CHANGELOG.md, README.md
```

**`tests/unit/helpers.py` is gold.** It contains ready-made curve builders
(`build_ibor_curve`, `build_full_issuer_curve`, ...). When you want to experiment with swaps or
CDS without typing 50 lines of market data, import from there.

---

## 4. Environment setup (do this first)

### 4.1 What I found in your current environment

| Item | Status |
|---|---|
| Python | 3.13.9 (Anaconda) |
| `import financepy` | ✅ works from the repo root |
| numpy 2.3.5 / scipy 1.16.3 / pandas 2.3.3 / matplotlib 3.10.6 | ✅ match `requirements.txt` |
| **numba 0.62.1** | ⚠️ **`pyproject.toml` requires `>=0.67.0,<0.68.0`.** Your global env is out of spec. |
| `pytest` at repo root | ❌ **fails before collecting a single test** (see §4.3) |
| git remotes | `origin` = your fork only. **No `upstream`** (see §8.1) |

### 4.2 Create a clean virtual environment (recommended)

Your global Anaconda env has the wrong Numba. Do not fight it — isolate:

```bash
cd /Users/efren/Desktop/quant_finance/FinancePy
python3 -m venv .venv
source .venv/bin/activate
python -m pip install --upgrade pip
pip install -r requirements.txt
pip install -e ".[test]"
python -c "import numba, financepy; print(numba.__version__, financepy.__version__)"
```

Also stop the repeated Matplotlib font-cache noise you will otherwise see on every run:

```bash
export MPLCONFIGDIR=/tmp/mplconfig && mkdir -p $MPLCONFIGDIR
```

### 4.3 Known rough edges (these are your future first PRs — see §9)

**A. `pytest` is currently broken at the repo root.** `pyproject.toml` contains:

```toml
filterwarnings = ["ignore::pyparsing.PyparsingDeprecationWarning"]
```

`pyparsing` 3.2.5 (installed) has no attribute `PyparsingDeprecationWarning`, so pytest aborts
while parsing its own config:

```
AttributeError: module 'pyparsing' has no attribute 'PyparsingDeprecationWarning'
```

Workaround until it is fixed upstream:

```bash
pytest tests/unit -q -o filterwarnings=
```

With that override, `tests/unit/test_FinEquityVanillaOption.py` passes: **4 passed in 9.71s**.

**B. Project documentation has drifted.** `README.md`'s "Structure of financepy" section shows
`discount_curve_flat.py`, but the real file is `market/curves/flat_discount_curve.py`; it also
shows capitalised product folders (`products/Bonds/`) when they are lowercase (`products/bonds/`),
and says "over 90 example notebooks" when there are 129. Not wrong in spirit, wrong in detail.

**C. `docs/` holds generated HTML only** (`find docs -name '*.md'` → 0 files), while `mkdocs.yml`
declares a nav built entirely from `.md` files (`index.md`, `quickstart.md`, `tutorials/bonds.md`,
...). The documented docs build cannot work as configured.

---

## 5. Math refresher map — what you use, where, and why

You have already met every prerequisite except stochastic calculus. This table is your lookup:
"which rusty concept do I need, and which file will make it concrete?"

| Math concept (from your courses) | Where it appears | FinancePy anchor |
|---|---|---|
| exp / log, exponents | compounding, discount factors, lognormal prices | `utils/compounding.py`, `market/curves/discount_curve.py` |
| Sums & series | annuities, swap fixed-leg PV, par rates | `products/bonds/bond_annuity.py`, `products/rates/swap_fixed_leg.py` |
| Interpolation | building curves through quoted points | `market/curves/interpolator.py`, `utils/tension_spline.py`, `utils/polyfit.py` |
| Root finding (bisection/Newton) | implied volatility, curve bootstrapping | `utils/solver_1d.py`; used in `black_scholes_analytic.implied_volatility` |
| Probability distributions | normal pdf/cdf, lognormal returns, expectations | `utils/math.py` (`normcdf`, `normpdf`, `norminvcdf`), `utils/distribution.py` |
| **Conditional expectation, risk-neutral pricing** | option value = discounted expected payoff | `models/equity_crr_tree.py`, `models/black_scholes_mc.py` |
| Multivariate normal, Cholesky | correlated asset paths, copulas | `utils/math.py::cholesky` & `corr_matrix_generator`, `models/gauss_copula*.py` |
| **Partial derivatives** | the Greeks (Δ, Γ, ν, Θ, ρ) are derivatives of V | `black_scholes_analytic.py` functions `delta/gamma/vega/theta/rho/vanna` |
| Taylor series | Newton-Raphson, small-parameter expansions | `utils/solver_1d.py::newton`; SABR/Hagan expansions |
| PDEs & finite differences | Black-Scholes PDE solved numerically | `models/finite_difference.py`, `models/finite_difference_psor.py` |
| Linear systems (tridiagonal) | implicit finite-difference solvers | `utils/math.py::solve_tridiagonal_matrix` |
| **Least squares / regression** | Longstaff-Schwartz American MC; curve fitting | `models/equity_lsmc.py`, `market/curves/curve_fits.py` |
| Optimisation | calibration to market quotes | `utils/solver_nm.py`, `utils/solver_cg.py` |
| Integration & transforms | Heston via characteristic functions | `models/heston.py` |
| Poisson processes | jump-diffusion credit/equity | `models/merton_jump_diffusion.py` |
| Hazard rate / survival probability | CDS pricing | `market/curves/cds_curve.py`, `discount_curve.survival_prob` |
| Copulas (dependence) | portfolio credit loss distributions | `models/gauss_copula_onefactor.py`, `student_t_copula.py` |
| **Stochastic calculus (NEW)** | GBM, Itô's lemma, SDEs, mean reversion | `models/gbm_process_simulator.py`, `vasicek_mc.py`, `cir_montecarlo.py`, `heston.py` |

**Where to add stochastic calculus:** you only truly need it at **Stage 9**, and only lightly
before that — Stage 4 uses the *result* (the Black-Scholes formula) and Stage 5–6 use the
*mechanics* (trees, simulation). Treat SDEs as "probability + calculus applied to a path" and it
will feel continuous with Calc 2 rather than alien.

---

## 6. The learning path — 13 stages, easy → hard

Estimates assume ~6–8 focused hours/week. **The order matters more than the pace.**

---

### Stage 1 — Plumbing: dates, day counts, schedules
**Tier A · Foundations · ~1 week · Math: none new**

**Goal.** Explain why a "1-year" period is not 1.0 year for every contract, and compute accrued
interest / year fractions by hand.

**Read (in order, they are small):**
1. `financepy/utils/compounding.py` (18 lines)
2. `financepy/utils/frequency.py` (57)
3. `financepy/utils/day_count.py` (301) — focus on `year_frac` and the `DayCountTypes` enum
4. `financepy/utils/schedule.py` (275)
5. `financepy/utils/date.py` (1053) — read `add_days`, `add_months`, `add_tenor`, `next_imm_date`, `next_cds_date` only. Skip the rest on first pass.

**Run:** `examples/notebooks/utils/FINDATE_CreatingAndManipulatingFinDates.ipynb`,
`FINDAYCOUNT_Introduction.ipynb`, `FINSCHEDULE_ExamplesOfScheduleGeneration.ipynb`.

**Exercise / acceptance test.** Verify these three facts with a script you write:
- `Date(19, 2, 2026).add_days(2)` prints `21-FEB-2026` (this is the README's own smoke test).
- Bullet (zero-coupon) accrual: `ACT_360` on 1 Jan → 1 Jul gives `181/360`, while `THIRTY_E_360`
  gives exactly `0.5`.
- Generate a 5-year semi-annual schedule and assert it has 10 payment dates.

**Why it matters.** Getting dates and day counts wrong is the #1 source of "my price is off by
2%" bug reports in every quant library. Build the habit now.

---

### Stage 2 — Discounting and interest-rate curves
**Tier A · Foundations · ~1–2 weeks · Math: exp/log, interpolation**

**Goal.** Discount cashflows at arbitrary dates; move between discount factors, zero rates,
compounded rates, and forward rates; explain what "bootstrapping" solves.

**Math to refresh.** Continuous vs periodic compounding
(`DF = (1 + r/n)^{-nT} = e^{-rT}` for continuous); forward rate from two discount factors;
linear vs log-linear vs cubic-spline interpolation.

**Read:**
1. `financepy/market/curves/flat_discount_curve.py` (small, subclass of `DiscountCurve`)
2. `financepy/market/curves/discount_curve.py` — read `df_t`, `zero_rate_cc`, `fwd_rate`,
   `zero_rate`, `par_rate`. This file is the spine of the whole library.
3. `financepy/market/curves/interpolator.py`
4. **Skim only:** `financepy/market/curves/ibor_single_curve.py` — read `_cost_function` and
   `_build_curve_using_1d_solver` to see bootstrapping as *root-finding*.

**Run:** `examples/notebooks/market/curves/FINDISCOUNTCURVE_Introduction.ipynb`,
`FINDISCOUNTCURVE_AnalysisOfInterpolationSchemes.ipynb`,
`FINNSDISCOUNTCURVE_ExaminingTheNelsonSiegelCurve.ipynb`.

**Exercise / acceptance test.**
- With `FlatDiscountCurve(value_dt, 0.05)`, assert `df_t(1.0) ≈ exp(-0.05)` and
  `df_t(2.0) ≈ exp(-0.10)`.
- Recover the continuously-compounded zero rate from a discount factor and get 0.05 back.
- **Derive by hand** the forward rate between years 1 and 2 implied by that flat curve and check it
  equals 0.05.

---

### Stage 3 — The math toolbox and root finding
**Tier A · Foundations · ~1 week · Math: probability, Taylor, linear algebra**

**Goal.** Know exactly which special functions the library provides and how it inverts functions —
because implied vol, calibration and bootstrapping are all root-finding.

**Read:**
1. `financepy/utils/math.py` — `normcdf`, `normpdf`, `norminvcdf`, `normcdf_prime`;
   `cholesky`, `corr_matrix_generator`; `solve_tridiagonal_matrix`.
2. `financepy/utils/solver_1d.py` — `newton`, `newton_secant`, `bisection`, `brent_max`.
3. `financepy/utils/stats.py` (106 lines) and `financepy/utils/distribution.py` (28).

**Run:** `examples/notebooks/utils/FINCALENDAR_IntroductionToUsingCalendars.ipynb` (light relief),
`examples/notebooks/models/FINGBMPROCESS_generatePaths.ipynb`.

**Exercise / acceptance test.** Write your own bisection solver in ~15 lines and use it to invert
`normcdf` (find `x` with `normcdf(x) = 0.975`; expect ≈ 1.959964). Then compare with
`financepy.utils.math.norminvcdf(0.975)`. This single exercise teaches you the exact machinery
used for implied volatility in Stage 4.

---

### Stage 4 — Black-Scholes and the European vanilla option ★ pivotal stage
**Tier B · Equity core · ~2–3 weeks · Math: probability + partial derivatives + first taste of Itô**

**Goal.** Derive the Black-Scholes formula, understand `d1`/`d2`, and derive every Greek as a
partial derivative. This is the stage where "concepts" turn into "math".

**Math to refresh.**
- Lognormal distribution: if `ln S_T ~ Normal(m, s²)`, then `E[max(S_T − K, 0)]` is an integral of
  the normal density → the two terms of the BS formula.
- **Risk-neutral pricing:** `V = e^{-rT} E_Q[payoff]` — the expectation is taken under a modified
  probability, and `r` replaces the drift. (This is the one idea to sit with.)
- Derivatives: `Δ = ∂V/∂S`, `Γ = ∂²V/∂S²`, `ν = ∂V/∂σ`, `Θ = −∂V/∂T`, `ρ = ∂V/∂r`.

**Read:**
1. `financepy/models/black_scholes_analytic.py` — **`european_value`** (around line 51) is the
   whole model in ~20 lines. Then `delta`, `gamma`, `vega`, `theta`, `rho`, `vanna`,
   `implied_volatility`.
2. `financepy/models/black_scholes.py` — the dispatcher that chooses analytic / tree / MC / FD.
3. `financepy/products/equity/equity_vanilla_option.py` — a clean template for *every* product
   class in the repo.

**Run:** `examples/notebooks/products/equity/EQUITY_VANILLA_EUROPEAN_STYLE_OPTION.ipynb`,
then `EQUITY_VANILLA_OPTION_ValuationAndGreeks.ipynb`, then
`examples/scripts/equity/example_equity_vanilla_option.py`.
Read the test: `tests/unit/test_FinEquityVanillaOption.py`.

**Exercise / acceptance test (the most important exercise in this file).**
Write `my_black_scholes.py` from scratch using only `math` — no FinancePy — for
`S = K = 100`, `r = 0.05`, `q = 0.01`, `σ = 0.20`, `T = 366/365` (note: FinancePy computes this
period as **1.002739726** because ACT/365F counts the leap-day year; that detail is itself a lesson
from Stage 1). You should reproduce:

| Quantity | FinancePy (verified) | Your derivation |
|---|---|---|
| call value | 9.84198085 | ≈ 9.84199281 |
| delta | 0.61190133 | ≈ 0.61190140 |
| vega | 37.80528689 | 37.80528689 |

(tiny differences are Numba `fastmath` rounding). **Then** explain to yourself why the call
increases with `σ` and decreases with `K`, using the sign of the derivative you wrote.

**Then** check put–call parity: `C − P = S e^{-qT} − K e^{-rT}`.

---

### Stage 5 — Binomial trees and risk-neutral valuation
**Tier B · Equity core · ~2 weeks · Math: expectation, backward induction, limits**

**Goal.** Price the *same* option three ways (formula, tree, simulation) and show they agree as the
tree refines — this is how you *believe* the theory instead of memorising it.

**Math to refresh.** Expectation as a weighted sum; working backwards through time; the idea that a
tree converges to the continuous GBM as `N → ∞`.

**Read:**
1. `financepy/models/equity_crr_tree.py` — `crr_tree_val` and `crr_tree_val_avg`. Follow the
   `p` (risk-neutral up-probability), the up/down factors, and the backward pass.
2. `financepy/products/equity/equity_american_option.py` (early exercise = compare intrinsic vs
   continuation at every node).
3. `financepy/products/equity/equity_binomial_tree.py` (the product-level tree wrapper).

**Run:** `examples/notebooks/products/equity/EQUITY_VANILLA_AMERICAN_STYLE_OPTION.ipynb`,
`examples/scripts/equity/example_equity_binomial_tree.py`,
`example_equity_american_option_crr_tree.py`.

**Exercise / acceptance test.** Plot European call price from the CRR tree for
`N = 10, 50, 100, 500, 2000` steps. Assert it converges monotonically-ish toward the analytic
Stage-4 value, and that the American put is worth **more** than the European put (early exercise
premium) when `r > 0`. Explain *why* in one sentence.

---

### Stage 6 — Monte Carlo: turning probability into numbers
**Tier B · Equity core · ~2 weeks · Math: CLT, standard error, variance reduction**

**Goal.** Simulate GBM paths, price a European option, and quantify your own error. This stage
overlaps most with your data-science instincts — and the repo has a whole file demonstrating
Numba speed-ups on the same problem.

**Math to refresh.** Law of large numbers; standard error `σ/√N`; antithetic variates
(use `Z` and `−Z`); quasi-random (Sobol) sequences instead of pseudo-random.

**Read:**
1. `financepy/models/gbm_process_simulator.py` — exact GBM discretisation.
2. `financepy/models/black_scholes_mc.py` — **read all six implementations side by side**:
   `value_mc_nonumba_nonumpy` → `value_mc_numpy_only` → `value_mc_numpy_numba` →
   `value_mc_numba_only` → `value_mc_numba_noanti` → `value_mc_numba_parallel`.
   This is the best Numba tutorial you will find for this domain.
3. `financepy/models/sobol.py`, `financepy/models/process_simulator.py`.

**Run:** `examples/notebooks/models/FINGBMPROCESS_generatePaths.ipynb`,
`examples/notebooks/products/equity/EQUITY_VANILLA_EUROPEAN_STYLE_MONTE_CARLO_SOBOL.ipynb`,
`EQUITY_VANILLA_EUROPEAN_STYLE_MONTE_CARLO_TIMINGS.ipynb`,
`examples/scripts/models/example_gbm_process.py`.

**Exercise / acceptance test.** Write a plain-NumPy MC pricer. For `N = 10³, 10⁴, 10⁵, 10⁶` record
the price and the standard error and **verify the error falls roughly like `1/√N`** (doubling
simulations, i.e. 4× work, halves the error). Then add antithetic variates and show the error drops
further at the same `N`.

---

### Stage 7 — American options and numerical methods (your data-science bridge)
**Tier B · Equity core / numerics · ~2–3 weeks · Math: PDE, discretisation, regression**

**Goal.** Understand the two workhorse numerical techniques beyond trees: finite differences (PDE)
and Longstaff-Schwartz (regression MC).

**Math to refresh.** The Black-Scholes **PDE**; discretising derivatives with finite differences
(`∂V/∂t`, `∂V/∂S`, `∂²V/∂S²` → `fn_dx`, `fn_dxx`); the θ-scheme (explicit/implicit/Crank-Nicolson);
solving a tridiagonal system; least-squares regression.

**Read:**
1. `financepy/models/finite_difference.py` — `fn_dx`, `fn_dxx`, `calculate_fd_matrix`,
   `fd_roll_backwards`, `option_payoff`.
2. `financepy/models/finite_difference_psor.py` — PSOR = projected successive over-relaxation, the
   trick for American early exercise.
3. `financepy/models/equity_lsmc.py` — Longstaff-Schwartz: regress continuation value on basis
   functions of the current state (this is literally regression, your home turf).
4. `financepy/utils/math.py::solve_tridiagonal_matrix`, `transpose_tridiagonal_matrix`.

**Run:** `examples/notebooks/models/FINITE_DIFFERENCE.ipynb`, `FINITE_DIFFERENCE_PSOR.ipynb`,
`examples/scripts/equity/example_equity_american_mc.py`.
Read `tests/unit/test_finite_difference.py`, `test_lsmc.py`.

**Exercise / acceptance test.** Price an American put with (a) CRR tree, (b) PSOR finite
difference, (c) LSMC. They should agree within a few basis points. Report the early-exercise
premium versus the European put. **Then reduce volatility to ~0** and check all three converge to
the intrinsic value — the repo's own unit-test philosophy ("only test edge cases that can never
drift") in action.

---

### Stage 8 — Exotic payoffs (reuse, mostly new formulas rather than new math)
**Tier B · Equity/FX · ~2–3 weeks · Math: reflection principle, path dependence**

**Goal.** See how small changes in the payoff contract radically change the mathematics, while the
product/model/market architecture stays identical.

**Read (one file per payoff; follow the same pattern as Stage 4):**
- `products/equity/equity_digital_option.py` + `models/bs_digital_option.py`
- `products/equity/equity_barrier_option.py` + `models/barrier_option_model.py` /
  `barrier_option_mc.py` (8 knock-in/knock-out variants)
- `products/equity/equity_asian_option.py` + `models/equity_asian_option_bs.py` /
  `equity_asian_option_mc.py` (average price → no simple closed form)
- `products/equity/equity_fixed_lookback_option.py`, `equity_float_lookback_option.py`
- `products/equity/equity_chooser_option.py`, `equity_compound_option.py`
- `products/equity/equity_one_touch_option.py`, `products/fx/fx_one_touch_option.py`
- `products/equity/equity_variance_swap.py`

**Run:** the matching notebooks —
`EQUITY_DIGITALOPTION_BasicValuation.ipynb`, `EQUITY_BARRIER_OPTIONS.ipynb`,
`EQUITY_ASIAN_OPTIONS.ipynb`, `EQUITY_FIXED_LOOKBACK_OPTION.ipynb`,
`EQUITY_CHOOSER_OPTION.ipynb`, `EQUITY_COMPOUND_OPTION_CompareWithML.ipynb`,
`FX_BARRIER_OPTIONS.ipynb`.

**Exercise / acceptance test.** Prove a model property numerically:
- A **digital call** ≈ `−∂(vanilla call)/∂K` — compute the vanilla delta w.r.t. `K` by finite
  difference and compare to the digital price.
- A **knock-out barrier** set very far away should equal the vanilla price; a knock-in plus
  knock-out should sum to the vanilla price (`KI + KO = vanilla`).

---

### Stage 9 — Stochastic volatility, jumps, and the volatility surface
**Tier C · Advanced models · ~4–6 weeks · Math: SDEs, Itô, Fourier, asymptotics**

**Goal.** Move from "constant σ" to the models actually used on trading desks. **This is where you
learn genuine stochastic calculus** — budget real time for it, and treat it as the summit, not a
checkpoint.

**Math to refresh / learn.**
- Brownian motion, SDEs, **Itô's lemma**, GBM as the solution of an SDE, mean reversion (OU process).
- Characteristic functions and Fourier inversion (Heston has no elementary closed form).
- Small-parameter asymptotic expansion (SABR's implied-vol formula — use Hagan's result first,
  derive later).
- Poisson jump processes; local volatility (Dupire) as a function of the whole surface.

**Read (in this order — hardest last):**
1. `financepy/models/cev.py` (592) — local vol as a power of `S`; a gentle SDE warm-up.
2. `financepy/models/merton_jump_diffusion.py` (964) — jumps; `merton.py` example in `models/`.
3. `financepy/models/heston.py` (616) — stochastic variance. Read the SDE, then `HestonValueTypes`
   (the enum listing the numerical methods).
4. `financepy/models/sabr.py` (315 lines), `sabr_shifted.py` — the desk-standard smile model.
   Hagan's implied-volatility formula is an asymptotic expansion: use it as a black box first,
   then come back and read the expansion terms.
5. `financepy/models/dupire.py` (531) — local volatility.
6. `financepy/models/svi.py`, `svi_surface.py`, `ssvi_surface.py`, `volatility_fns.py`,
   `implied_volatility_surface.py` — how a *surface* is parameterised and arbitrage-controlled.
7. `financepy/market/volatility/equity_vol_surface.py`, `fx_vol_surface.py`.

**Run:** `examples/notebooks/models/MERTON_CREDIT_MODEL.ipynb`,
`FINMODEL_SABR_InterestRates.ipynb`, `FINMODEL_SABRSHIFTED_VolatilitySmile.ipynb`,
`FINVOLFUNCTIONS_SSVI_MODEL.ipynb`,
`examples/notebooks/market/volatility/EquityVolSurfaceConstructionSVI.ipynb`,
`FXVolSurfaceConstructionPartOne/Two/Three.ipynb`,
`examples/scripts/models/example_model_heston.py`, `example_model_merton_jump_diffusion.py`.

**Exercise / acceptance test.** A *limit* test (very much in the repo's spirit):
- In Heston, set vol-of-vol `= 0` and correlation `= 0`; the price must collapse to
  Black-Scholes with `σ = √v₀`. Verify numerically across several strikes.
- In CEV, set the exponent `= 1`; it must collapse to Black-Scholes.

A limit test proves you understood the model's degrees of freedom, not just its API.

---

### Stage 10 — Interest-rate products, trees, and calibration
**Tier D · Rates · ~4–6 weeks · Math: curves (deeper), trinomial trees, calibration**

**Goal.** Price swaps, caps/floors and swaptions; understand why multi-curve discounting exists and
how arbitrage-free rate trees are calibrated.

**Math to refresh.** Present value of a swap = difference of two legs; par rate; forward-rate
bootstrapping (Stage 2, now for real instruments); trinomial trees and backward induction;
calibration as optimisation (Stage 3 toolbox); multi-curve (OIS discounting vs IBOR forecasting).

**Read:**
1. `products/rates/ibor_deposit.py`, `ibor_fra.py`, `ibor_future.py` (small, start here)
2. `products/rates/swap_fixed_leg.py`, `swap_float_leg.py`, `ibor_swap.py`
3. `market/curves/ibor_single_curve.py` (now read `_precompute_swap_flows`,
   `_build_curve_using_1d_solver`, `check_refit` properly)
4. `market/curves/ois_curve.py`, `products/rates/ois.py`, then `dual_curve.py` /
   `products/rates/ibor_dual_curve.py` (multi-curve)
5. `products/rates/ibor_cap_floor.py`, `ibor_swaption.py`, `ibor_bermudan_swaption.py`
6. Trees: `models/hw_tree.py` (1485 — Hull-White), `models/bk_tree.py` (1238), `models/bdt_tree.py`
7. `models/lmm_mc.py` (1052 — LIBOR Market Model, multi-factor Monte Carlo)
8. `market/volatility/swaption_vol_surface.py`, `ibor_cap_vol_curve.py`

**Run:** `examples/notebooks/products/rates/FINIBORSWAP_DefiningAFixedFloatingSwap.ipynb`,
`FINIBORSWAP_ComparisonWithQLExample.ipynb`,
`examples/notebooks/market/curves/FINIBORSINGLECURVE_BuildingASimpleIborCurve.ipynb`,
`FINIBORDUALCURVE_BuildingASimpleDualIborCurve.ipynb`,
`examples/notebooks/products/rates/FINIBORSWAPTION_ValuingAcrossModels.ipynb`,
`FINIBORCAPFLOOR_ValuationAcrossAllModels.ipynb`,
`examples/scripts/rates/`, and `tests/unit/helpers.py::build_ibor_curve`.

**Exercise / acceptance test.**
- Build a curve from `tests/unit/helpers.build_ibor_curve` and assert each input swap prices to
  **≈ 0** (that is what "calibrated" means). This is exactly what
  `IborSingleCurve.check_refit` verifies.
- Price a swaption with Black and with Hull-White trinomial tree; they should be close at the
  money and diverge away from it (the smile).

---

### Stage 11 — Credit derivatives: default, hazard rates, dependence
**Tier D · Credit · ~3–5 weeks · Math: hazard rates, Poisson, copulas**

**Goal.** Price a CDS, build a survival curve, then understand portfolio credit (correlation,
tranches, copulas).

**Math to refresh.** Hazard rate / intensity: `survival(t) = exp(−∫λ)`, default probability, the
recovery rate and the CDS "protection − premium" decomposition; Poisson processes; copulas and
conditional independence; loss distributions.

**Read:**
1. `products/credit/cds.py`, then `market/curves/cds_curve.py` (survival curve bootstrapping —
   structurally the same as Stage 2/10, so it will feel familiar).
2. `financepy/market/curves/discount_curve.py::survival_prob`.
3. `models/merton_firm.py`, `merton_firm_mkt.py` — Merton's structural model: **equity is a call
   option on the firm's assets.** This beautifully reuses Stage 4.
4. `models/gauss_copula_onefactor.py`, `gauss_copula.py`, `gauss_copula_lhp.py`,
   `gauss_copula_lhplus.py`, `student_t_copula.py`, `loss_dbn_builder.py`.
5. `products/credit/cds_tranche.py`, `cds_index_option.py`, `cds_option.py`, `cds_basket.py`.

**Run:** `examples/notebooks/products/credit/FINCDS_CreatingAndValuingACDS.ipynb`,
`FINCDS_CreatingAndValuingACDSFlatCurves.ipynb`,
`examples/notebooks/market/curves/FINCDSCURVE_BuildingASurvivalCurve.ipynb`,
`examples/notebooks/models/MERTON_CREDIT_MODEL.ipynb`,
`FINMODEL_GAUSSIANCOPULA_PortfolioLossDistributionBuilder.ipynb`,
`examples/notebooks/products/credit/FINCDSTRANCHE_CalculatingFairSpread.ipynb`.

**Exercise / acceptance test.**
- Build a survival curve and assert the input CDS contracts price to ≈ 0 (par), as in Stage 10.
- For a tranche: check that an "equity" tranche (attachment 0) is worth more than a "senior"
  tranche, and that **increasing default correlation moves value from the equity tranche to the
  senior tranche** (the central intuition of the 2008 crisis). Explain it in one sentence.

---

### Stage 12 — Bonds and their embedded options
**Tier D · Bonds · ~2–3 weeks · Math: mostly reuse (duration/convexity are derivatives)**

**Goal.** Tie the repo back to what you already own in your portfolio, and see how trees price a
callable bond.

**Math to refresh.** Price–yield relationship; **duration = −(1/P)·∂P/∂y** and
**convexity = (1/P)·∂²P/∂y²** (these are the same "Greeks are derivatives" idea from Stage 4);
spread measures (z-spread, asset-swap spread, OAS); prepayment modelling.

**Read:** `products/bonds/bond.py` (duration, convexity, spreads),
`bond_zero.py`, `bond_frn.py`, `cashflow.py`, `bond_annuity.py`, `bond_convertible.py`,
`bond_option.py`, `bond_inflation.py`, `bond_mortgage.py`, `bond_future.py`, `bond_portfolio.py`;
then `models/bond_convertible_bs_tree.py` and the `hw_tree`/`bk_tree` used by `bond_option`.

**Run:** `examples/notebooks/products/bonds/FINBOND_ExampleUSTreasury_CUSIP_91282CFX4.ipynb`,
`FINBOND_CalculatingTheAssetSwapSpread.ipynb`, `FINBOND_CalculateOptionAdjustedSpread.ipynb`,
`FINBOND_Key_Rate_Durations_Example.ipynb`, `FINBONDMORTGAGE_SimpleCalculator.ipynb`,
`FINBONDCONVERTIBLE_ValuationAndConvergenceTest.ipynb`,
`examples/notebooks/products/bonds/FINBONDMARKET_DatabaseOfConventions.ipynb`.

**Exercise / acceptance test.** Two of the repo's own canonical edge cases:
- A bond with coupon = yield must price at **exactly 100**.
- Compute duration numerically as `−(P(y+h) − P(y−h)) / (2h·P)`, and compare with FinancePy's
  analytic `bond.duration(...)`.
- Price a convertible bond as the straight bond + a call option on the stock; sanity-check the
  components.

---

### Stage 13 — Capstone: build something, then contribute
**Ongoing**

By now you will have opinions. Good capstone projects that prove mastery *and* produce useful
artifacts:
- Replicate a notebook's results independently (from the formula, not by calling the library) and
  write up the differences.
- Compare FinancePy against QuantLib on a product the repo's notebooks already reference (several
  notebooks are explicitly "ComparisonWithQLExample" — extend one).
- Add a unit test for an edge case you discovered (see §8.4) — this is also the *ideal* first PR.

---

## 7. Python-for-FinancePy skill checklist

You said Python is "slightly weak, up to OOP". That is enough to start Stage 1, and these are the
only extra things you need. Learn them *just in time* as you hit them, not upfront.

| Skill | Where you meet it | Why it matters here |
|---|---|---|
| Type hints (`Union`, `List`, `float \| np.ndarray`) | everywhere in recent files | Half the "what is this argument?" confusion disappears |
| `Enum` classes (`OptionTypes`, `DayCountTypes`) | `utils/global_types.py` | This is how the library encodes products/conventions |
| NumPy arrays & **vectorisation** | `black_scholes_analytic.py`, all MC | Whole vectors of strikes/expiries priced in one call |
| Classes & inheritance | `DiscountCurve` → `FlatDiscountCurve`; `Model` base | Understand what a subclass may override (`df_t`) |
| **Numba decorators** (`@njit`, `@vectorize`, `fastmath`, `cache`) | most of `models/`, `utils/math.py` | Explains the import delay (first-call compile) and the tiny numeric differences vs your hand maths |
| `pytest` basics | `tests/unit/` | How you contribute tests |
| Reading a class in 5 steps | any product file | The transferable skill of this repo — see recipe below |

**How to read any FinancePy product class (the 5-step recipe):**
1. Read the class docstring — it states the payoff in English.
2. Read `__init__` — that is the *data* defining the contract (dates, strike, type, notional).
3. Read `value(...)` — notice it takes `market` objects and a `model`, then usually
   calls **one function** in `models/`.
4. Jump to that model function and read the formula.
5. Find the matching `tests/unit/test_...py` and the `examples/...` notebook/script.
If you can do those five steps on a product you have never seen, you have learned the repo.

---

## 8. Contribution guide (secondary, but do it steadily)

### 8.1 Sync your fork to upstream first

Your fork currently has **only** `origin` (your fork); there is no `upstream` remote, so your
`master` can silently drift. I verified upstream's head is `cc14000b8c8b95bbe44e7af9a871caf1497108e1`
while your local `master` is at `c2dff829` — **confirm the gap before doing anything else:**

```bash
git remote add upstream https://github.com/domokane/FinancePy.git
git fetch upstream
git log --oneline --left-right upstream/master...master | head   # see how far apart you are
```

### 8.2 Branch workflow (keeps your personal files out of PRs)

```bash
git checkout master
git merge upstream/master          # or: git rebase upstream/master
git checkout -b fix/short-description   # one branch per contribution
# ... work, commit ...
git push -u origin fix/short-description
# open the PR against domokane/FinancePy:master (NOT your master)
```

`LEARNING_ROADMAP.md` this file, and any scratch scripts stay on `master`. Feature branches are
created *from* `master` but should only contain the contribution. If you want to be extra safe, put
scratch work in an ignored folder, e.g. add `learning_scratch/` to `.gitignore`.

### 8.3 The repo's own contribution rules (from README)

- PEP8 compliant (`.pylintrc` and a `pylint.yml` workflow enforce this).
- **A docstring on every class and function**, with a clear description.
- **At least one broad test case plus unit tests for every function.**
- **Avoid "very pythonic" constructions** — an explicit loop is preferred to a list comprehension,
  and it is often faster under Numba. Readability and speed are the stated priorities.

### 8.4 The two test suites — know which one to use

| | `tests/unit/` (pytest) | `tests/regression/` (golden files) |
|---|---|---|
| Purpose | deterministic, never-drifting edge cases | full-parameter model output vs stored golden values |
| Examples | zero vol, put-call parity, deep ITM/OTM, par bond | every strike × step count × model combination |
| Tool | `pytest` | `FinTestCases` + `compare_test_cases()` |
| Docs | `tests/unit/REAME.md` | `tests/regression/README.md` (read it — it is clear) |

Rule of thumb: put **new functionality** and **edge cases** in `tests/unit/`; only touch the golden
files in `tests/regression/` when underlying model output legitimately changes, and say so in the
PR. The regression tolerance is `1e-8`; `TIME`-labelled columns warn instead of error.

### 8.5 PR etiquette (from README + observed practice)

1. Find or file an issue, and **comment in the thread** to claim it (avoids duplicate work).
2. Keep PRs small and single-purpose; big mixed PRs stall.
3. Include the exact reproduction command and expected vs actual output.
4. Run `pytest tests/unit -q -o filterwarnings=` before pushing (see §4.3A).
5. The maintainer is Dominic O'Kane (`dominic.okane at edhec.edu`). Be concise and specific.

---

## 9. Verified first-contribution candidates (easiest → hardest)

All four below were **confirmed by running commands in this repo**, so none is speculative.

**① Fix the broken pytest configuration** — *~15 minutes, high value, blocks every contributor*
- **Problem:** `pytest` fails before collecting any test:
  `AttributeError: module 'pyparsing' has no attribute 'PyparsingDeprecationWarning'`
- **Cause:** `pyproject.toml` → `[tool.pytest.ini_options] filterwarnings` references a pyparsing
  warning class that does not exist in installed pyparsing 3.2.5.
- **Evidence:** reproduced; `-o filterwarnings=` makes tests pass (`4 passed in 9.71s`).
- **Fix:** remove the entry or make it version-safe.
- **Note:** also check `requirements.txt` should probably pin `pyparsing` (or the filter should not
  depend on it). This is a genuine CI/developer-experience bug.

**② Fix the stale `README.md` structure section** — *~30 min*
- Shows `discount_curve_flat.py`; real file is `market/curves/flat_discount_curve.py`.
- Shows `products/Bonds/` / `Bond/`; real folders are lowercase `bonds`, `credit`, `equity`, `fx`,
  `rates`.
- Says "over 90 example notebooks"; there are **129** (`find examples/notebooks -name '*.ipynb' | wc -l`).
- Drive-by: the `models/` listing omits many current files.

**③ Fix the filename typo `tests/unit/REAME.md` → `README.md`** — *~5 min, warm-up PR*
- The file also contains typos ("unti testing"), and says `pytest --ignore legacy` while
  `pyproject.toml` uses `addopts = ["--ignore=tests/unit/legacy"]` — and **no `tests/unit/legacy`
  directory exists at all** (`test -d tests/unit/legacy` → false). Reconcile both the filename and
  the instructions.
- Renaming a file is a perfect first PR: trivial, clearly correct, forces you through the full
  fork → branch → PR loop.

**④ Reconcile `docs/` with `mkdocs.yml`** — *hours, needs a maintainer discussion first*
- `mkdocs.yml` declares a nav of `.md` files (`index.md`, `quickstart.md`, `tutorials/*.md`,
  `api/*.md`) but `docs/` contains **zero** `.md` files — only generated HTML.
- Either the `mkdocs.yml` nav is aspirational (dead config) or those sources were never committed.
  **Open an issue asking which**, then help. Do not guess at a big docs restructure.

**⑤ Add a missing edge-case unit test** — *the ideal "prove you learned it" PR*
- Once you reach Stage 4–7, you will find limit cases worth locking down (many are listed as
  exercises above: zero-vol collapse, put-call parity, calibibration-to-par, copula monotonicity).
- Match the style of `tests/unit/test_FinEquityVanillaOption.py`.

> Do ①②③ to learn the mechanics with near-zero risk, then spend your real effort on ⑤.

---

## 10. Progress tracker

**Foundations**
- [ ] Environment venv created, `pip install -e ".[test]"` succeeds
- [ ] `pytest tests/unit -q -o filterwarnings=` runs green
- [ ] Stage 1 — dates / day counts / schedules · acceptance test passes
- [ ] Stage 2 — discount curves · acceptance test passes
- [ ] Stage 3 — math toolbox + own bisection solver · acceptance test passes

**Equity derivatives core**
- [ ] Stage 4 — **BS derived by hand, matches 9.8420 / 0.6119 / 37.8053** ★
- [ ] Stage 5 — tree ↔ formula convergence
- [ ] Stage 6 — MC error ∝ 1/√N
- [ ] Stage 7 — American: tree ≈ FD ≈ LSMC
- [ ] Stage 8 — exotics: digital ≈ −∂call/∂K, KI + KO = vanilla

**Advanced & other asset classes**
- [ ] Stage 9 — stochastic calculus; Heston → BS limit test
- [ ] Stage 10 — swaps calibrated to par; swaption Black vs HW
- [ ] Stage 11 — CDS par; correlation shifts tranche value
- [ ] Stage 12 — bond at par when coupon = yield; duration by finite difference
- [ ] Stage 13 — capstone written up

**Contribution**
- [ ] `upstream` remote added and `master` synced
- [ ] First PR merged (candidate ③ or ②)
- [ ] Second PR merged (candidate ①)
- [ ] Own edge-case unit test merged (candidate ⑤)

---

## 11. Companion resources (use *alongside*, not instead of)

- **The repo's own docs:** `https://domokane.github.io/FinancePy/` (API reference),
  `examples/notebooks/` (129 notebooks — your primary teacher),
  `CHANGELOG.md` (shows what is actively changing; useful for finding work).
- **Textbooks that match this repo's level and scope:**
  - Hull, *Options, Futures and Other Derivatives* — the standard mapping to Stages 4–10
    (trees, BS, Greeks, swaps, credit).
  - Hull, *Risk Management and Financial Institutions* — lighter, good for Stages 11–12.
  - Wilmott / O'Kane's own *Modelling Single-name and Multi-name Credit Derivatives* (the
    maintainer's book) — for Stage 11 and beyond.
- **For the missing stochastic calculus (Stage 9):** any short introduction to SDEs and Itô's
  lemma; combine it with the *limit tests* above, which will keep you honest.

---

*Verify, don't trust: every time this file states a number or a fact, I ran it. Keep that habit —
it is the entire difference between reading about quant finance and being able to do it.*
