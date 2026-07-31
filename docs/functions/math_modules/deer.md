# DEER / PDS Distance Analysis

Distance-distribution analysis for pulsed-dipolar spectroscopy (DEER/PELDOR, and
the closely related RIDME / DQC / SIFTER). All of these share one model: a
background-corrected **form factor** $F(t)$ is the integral over the distance
distribution $P(r)$ of an orientation-averaged dipolar kernel,

$$
F(t) = \int K(t, r)\,P(r)\,dr ,\qquad
K(t, r) = \int_0^1 \cos\!\big[(1 - 3\xi^2)\,\omega(r)\,t\big]\,d\xi ,
$$

with the dipolar angular frequency $\omega(r) = 2\pi\,\nu_{dd}/r^3$ (rad/µs, $r$
in nm, $t$ in µs) and $\nu_{dd} = 52.04\ \text{MHz·nm}^3$. The kernel integral
has a closed form in Fresnel integrals, so $K$ is built without a
per-orientation loop.

Recovering $P(r)$ from $F(t)$ is a Fredholm equation of the first kind
(ill-posed). This module solves it three ways:

- **Tikhonov regularization + non-negativity** (NNLS) — the default. The
  regularization weight $\alpha$ is chosen automatically (by **generalized
  cross-validation (GCV)** by default, or the classic **L-curve** corner). The
  background can be removed **sequentially** (fit the tail, divide it out,
  invert) or fit **jointly** with $P(r)$ (DeerLab-style). A covariance
  confidence band ([`tikhonov_ci()`](#tikhonov_ci)) is returned with every
  inversion.
- **Analytic integral Mellin transform** — a *model-free* inversion
  ([`deer_invert_mellin()`](#deer_invert_mellin), Matveeva, Nekrasov & Maryasov,
  *PCCP* **2017**, [10.1039/C7CP04059H](https://doi.org/10.1039/C7CP04059H)). No
  Tikhonov, no NNLS: $P(r)$ is recovered in closed form, so it is not broadened
  and bimodal peaks are not merged. Noise enters $P(r)$ additively and groups at
  short $r$.
- **Parametric sum-of-Gaussians fit** — a *model-based* inversion
  ([`deer_invert_gauss()`](#deer_invert_gauss); the DeerAnalysis "Gaussian" mode /
  DeerLab `dd_gaussN` approach). $P(r)$ is modelled as $N$ Gaussians fit jointly
  with the background to the signal, with $N$ chosen by an information criterion.
  When the distribution really is a few discrete modes this is the most robust choice and
  it gives genuine **parametric error bars** on each peak — including rigorous
  support-plane confidence intervals (Stein, Beth & Hustedt, *Methods Enzymol.*
  **2015**, [10.1016/bs.mie.2015.07.031](https://doi.org/10.1016/bs.mie.2015.07.031)).

The dipolar **zero-time** can be fit automatically with
[`fit_zero_time()`](#fit_zero_time) before any of these. The intermolecular
background is normally a stretched exponential ([`background_fit()`](#background_fit)),
but a flexible empirical form $a\,e^{\,b(t + c\,d^{\,t})}$ is also available
([`background_general()`](#background_general)).

!!! warning "Samples before the zero-time are dropped"
    Every engine discards $t < 0$ on entry. The kernel evaluates $|\omega t|$, so a
    negative-time sample would be modelled as ordinary evolution at $+|t|$, and the
    non-negative inversion then piles $P(r)$ mass at short $r$ to reproduce the
    echo's *rising edge* — reporting a distance far shorter than the true one, with
    no warning. Pass the whole trace anyway: the shift by `fit_zero_time()` and the
    crop are handled for you, and the returned `t`, `form_factor`, `F_fit` and
    background arrays are all defined on $t \ge 0$ only. Those samples are not
    wasted — they are what fixes the zero time — and the DEER tool still *draws*
    them, since $B(t)$ and $F(t) = K(|t|)P$ are exactly even and can be evaluated
    there without refitting.

!!! tip "Why GCV is the default"
    A DEER L-curve is nearly *vertical* — the residual stays at the noise floor
    across decades of $\alpha$ — so the Menger-curvature "corner" is ill-defined.
    Measured on the test traces, `method='curvature'` lands at **either end**
    of the grid depending only on where the background window starts.
    GCV has a single — if shallow — minimum, matches DeerLab's `gcv` selection
    exactly on a shared grid, and is used by default; treat `'curvature'` as a
    cross-check only. For a $P(r)$ with a trustworthy uncertainty band use
    [`deer_validate()`](#deer_validate), which averages over background choices.

!!! note "scipy is required"
    DEER analysis needs `scipy` (the `math` extra: `pip install -e .[math]`).
    scipy is imported lazily, so importing the module never fails on a minimal
    install — the routines raise a `RuntimeError` when scipy is missing.

!!! info "Conventions"
    Times are in **microseconds**, distances in **nanometres**. Internally
    $P(r)$ is handled as discrete probability masses (sum $= 1$); the matching
    density $P(r) = \text{masses}/dr$ is returned for plotting.

```python
import numpy as np
import atomize.math_modules.deer as deer
```

---

## deer_invert() { #deer_invert data-toc-label="deer_invert" }

```python
res = deer.deer_invert(t, V, r=None, bg_start=None, bg_end=None,
                       dim=3.0, fit_dim=False, alpha=None, alphas=None,
                       reg_order=2, nu_dd=deer.NU_DD, scan_lcurve=True,
                       method='gcv', engine='sequential', alpha_factor=1.0)
```

The full one-call pipeline: background-correct $V(t)$, build the kernel, and
invert to $P(r)$ by Tikhonov + NNLS. This is what most users want.

- **`t`, `V`** — time axis (µs) and the real DEER trace $V(t)$.
- **`r`** — distance grid (nm). `None` uses [`default_r_axis()`](#default_r_axis)
  (1.5–8 nm, 200 points).
- **`bg_start`, `bg_end`** — background-fit window (µs). `bg_start=None` defaults
  to the midpoint of the trace; `bg_end=None` fits to the end. See
  [`background_fit()`](#background_fit). (With `engine='joint'` they set the
  **tail baseline window** for the λ-pinned, truncated-grid joint background fit —
  see [`deer_invert_joint()`](#deer_invert_joint).)
- **`dim`, `fit_dim`** — fractal background dimension (3 = homogeneous 3D); set
  `fit_dim=True` to float it.
- **`alpha`** — regularization weight. `None` selects it automatically by `method`.
- **`alpha_factor`** — multiplier applied to the *auto-selected* $\alpha$ (ignored
  when an explicit `alpha` is given). A factor of 2–4 reproduces the heavier
  hand-picked L-corner regularization used to obtain smooth distributions in
  inter-laboratory ring tests
  ([Schiemann et al., *JACS* **2021**, 143, 17875](https://doi.org/10.1021/jacs.1c07371)).
  It buys that smoothness with **bias**: $P(r)$ is pulled measurably off the truth
  while the [`tikhonov_ci()`](#tikhonov_ci) band, which propagates noise only, gets
  *narrower*. Measured coverage of the nominal-95% band at the mode falls from
  ≈ 0.84 at 1× to ≈ 0.08 at 2× and ≈ 0 at 3×. Above 1×, read the band as a noise
  scale rather than a confidence interval.
- **`alphas`** — the regularization scan grid (default `np.logspace(-4, 3, 36)`).
- **`reg_order`** — derivative order of the smoothing operator $L$ (default 2).
- **`scan_lcurve`** — when `True` (default) the regularization scan is always
  computed for display, even if an explicit `alpha` is given.
- **`method`** — automatic-$\alpha$ criterion: `'gcv'` (default — generalized
  cross-validation, robust) or `'curvature'` (classic maximum-Menger-curvature
  L-corner). See [`l_curve()`](#l_curve).
- **`engine`** — how the inversion is done:
  `'sequential'` (default; fit the background tail, divide it out, then invert),
  `'joint'` (fit background + modulation depth together with $P(r)$ in one pass —
  see [`deer_invert_joint()`](#deer_invert_joint); more robust when the background
  window is short or hard to place), `'mellin'` (the model-free analytic
  transform — see [`deer_invert_mellin()`](#deer_invert_mellin)), `'gauss'` (the
  parametric sum-of-Gaussians fit — see [`deer_invert_gauss()`](#deer_invert_gauss)),
  or `'none'` (**no background**: $B(t)=1$, fit only the modulation depth
  $\lambda$ — for pre-corrected / simulated / full-modulation $\lambda\!\to\!1$
  data; fitting a decay there would absorb the dipolar decay and badly broaden
  $P(r)$). `'general'` selects the empirical
  [`background_general()`](#background_general) background with an otherwise
  sequential Tikhonov inversion.
- **`**kwargs`** — forwarded to the model-free / parametric engines:
  `engine='mellin'` takes `delta`, `tau_max`, `n_tau`, `bg_engine`, `n_mc`, …;
  `engine='gauss'` takes `n_gauss`, `max_gauss`, `ic`, `ci_mode`, `bg_engine`, …;
  `bg_params` (the [`background_general()`](#background_general) coefficients) is
  forwarded to any engine. Ignored otherwise.

Returns a dict:

| Key | Description |
| --- | ----------- |
| `t`, `r` | The time and distance axes used |
| `form_factor` | Background-corrected form factor $F(t)$ |
| `F_fit` | Back-calculated fit $K P$ |
| `residuals` | `form_factor - F_fit` |
| `P` | Raw distance masses ($\ge 0$) |
| `P_norm` | Masses normalized to sum $= 1$ |
| `P_density` | Density $P(r) = $ `P_norm`$/dr$ (integral $= 1$) — plot this |
| `P_lower`, `P_upper` | 95% **noise-only** band on the density — not a calibrated CI, see [`tikhonov_ci()`](#tikhonov_ci) |
| `ci_kind` | `'noise'`, or `'noise_fixed_bg'` for `engine='joint'` (background and $\lambda$ held fixed) |
| `kernel` | The dipolar kernel matrix $K$ |
| `alpha` | The regularization weight used |
| `l_curve` | The [`l_curve()`](#l_curve) result dict (or `None`) |
| `background` | The [`background_fit()`](#background_fit) result dict |
| `lambda`, `k`, `dim` | Modulation depth, background decay rate, dimension |
| `engine` | `'sequential'`, `'joint'`, `'mellin'`, `'gauss'`, `'none'`, or `'general'` |

```python
import numpy as np
import atomize.math_modules.deer as deer

# synthetic 3.5 nm trace
r = deer.default_r_axis(2.0, 5.0, 150)
P = np.exp(-(r - 3.5)**2 / (2*0.15**2))
t = np.linspace(0, 3.0, 300)                       # us
V = deer.simulate(t, r, P, lam=0.3, k=0.1, dim=3.0, noise=0.01, seed=1)

res = deer.deer_invert(t, V, r=r, bg_start=1.0)
peak = res['r'][res['P_density'].argmax()]
print(f"lambda = {res['lambda']:.3f}, alpha = {res['alpha']:.3g}, peak r = {peak:.2f} nm")
```

---

## deer_invert_joint() { #deer_invert_joint data-toc-label="deer_invert_joint" }

```python
res = deer.deer_invert_joint(t, V, r=None, bg_start=None, bg_end=None,
                             dim=3.0, fit_dim=False, alpha=None, alphas=None,
                             reg_order=2, nu_dd=deer.NU_DD, method='gcv',
                             scan_lcurve=True, alpha_factor=1.0)
```

DEER inversion with a **joint** fit of the background and modulation depth
*together* with the regularized non-negative $P(r)$ — the strategy DeerLab uses.
More robust than the sequential [`deer_invert()`](#deer_invert) pipeline on real
traces with short or shallow backgrounds, where the tail fit and the inversion
are coupled. Also reachable as `deer.deer_invert(..., engine='joint')`.

Starting from the full model

$$
V(t) = B(t)\,\big[(1-\lambda) + \lambda\,(K P)(t)\big],\qquad
B(t) = e^{-(k|t|)^{d/3}},
$$

the only nonlinear unknown is the background decay rate $k$ (and $d$ when
`fit_dim=True`). The background and modulation depth are fit by
[`joint_background()`](#joint_background), the same fit the Mellin engine uses.
$\lambda$ is pinned to the tail baseline of $V/B$ over
$[\text{bg\_start}, \text{bg\_end}]$ (where the form factor has decayed and
$V \approx (1-\lambda)\,B$), and $k$ is fit together with a coarse non-negative
$P(r)$ on a distance grid truncated at the supported $r_\text{max}$. The
full-resolution $P(r)$ then follows from $K P = (V/B - (1-\lambda))/\lambda$ by
Tikhonov + NNLS, with $\alpha$ chosen by GCV as in [`deer_invert()`](#deer_invert).

!!! note "Why the rate is fit on a truncated grid"
    This breaks two coupled ambiguities. First, on the full $r$ grid a gentle
    background can be imitated by spurious long-distance $P(r)$ mass, so an
    unconstrained rate search collapses to $k \to 0$ and broadens $P(r)$; truncating
    the grid at $r_\text{max}$ removes that escape route. Second, with $\lambda$
    free, a shallow background plus extra long-$r$ mass can also imitate the correct
    deeper background, so $\lambda$ is pinned to the decayed-tail baseline. So
    `bg_start`/`bg_end` set the **baseline window**, not just an initial guess.

Returns the same dict as [`deer_invert()`](#deer_invert), with `engine='joint'`.

!!! tip "Lightweight variant for the Mellin engine"
    [`joint_background()`](#joint_background) runs the same λ-pinned rate fit but
    returns **only** the background (no full-resolution inversion / L-curve) on a
    coarse internal grid, and is further hardened against collapse on short
    traces / short `bg_end`. It is what [`deer_invert_mellin()`](#deer_invert_mellin)
    and Mellin validation use.

---

## tikhonov_ci() { #tikhonov_ci data-toc-label="tikhonov_ci" }

```python
lower, upper = deer.tikhonov_ci(K, F, alpha, P, L=None, dr=1.0, z=1.96)
```

Pointwise **noise-propagation band** on the regularized $P(r)$, returned with every
Tikhonov inversion.
For the linear Tikhonov estimator $P = (K^\top K + \alpha^2 L^\top L)^{-1} K^\top F$
the form-factor noise propagates as

$$
\operatorname{cov}(P) = \sigma^2\, M M^\top,\qquad
M = (K^\top K + \alpha^2 L^\top L)^{-1} K^\top,
$$

with $\sigma^2$ estimated from the fit residuals (effective dof
$= N - \operatorname{tr}(K M)$). Returns `(lower, upper)` at confidence `z`
(default 1.96 ≈ 95%) on the same density scale as `P/sum(P)/dr`, clipped at 0.

!!! warning "This is not a calibrated confidence interval"
    The band propagates **noise only**. It excludes the regularization bias, which
    is the dominant error at the peaks, and it is not DeerLab's band.
    Treat it as a display aid for the noise level. For a coverage-honest interval use
    [`deer_validate()`](#deer_validate) or the Mellin / multi-Gaussian Monte-Carlo
    bands.

---

## fit_zero_time() { #fit_zero_time data-toc-label="fit_zero_time" }

```python
t0 = deer.fit_zero_time(t, V, bg_start=None, bg_end=None,
                        n_grid=16, search_frac=0.15, refine=True,
                        method='parabola', drop=0.15, smooth_w=5,
                        xcheck=False, xcheck_tol_frac=0.004, **kwargs)
```

Find the dipolar **zero-time** $t_0$ (the reference time). DEER is sensitive to
where $t = 0$ of the dipolar evolution sits: an error of even a few tens of ns
misaligns the kernel, broadens $P(r)$ and biases the mean distance long. This is
the equivalent of DeerLab's fitted `reftime`, and it matters more than the
background depth.

Two methods, selected by **`method`**:

- **`'parabola'` (default)** — fit a quadratic to the **echo maximum** (the
  classic DeerAnalysis approach: $V \approx V_\text{pk} - c\,(t-t_0)^2$ near the
  echo) and take its vertex. Noise-robust: the initial peak is the argmax of a
  *smoothed* $V$ restricted to the first 30 % of the trace (so a stray noise spike
  cannot be mistaken for the echo), and the fit window widens symmetrically out to
  where the smoothed signal has fallen `drop` of its peak-to-min amplitude — wide
  enough to average down noise, narrow enough to stay on the parabolic top (a
  too-wide window is biased by the dipolar oscillation/decay and the truncated
  pre-zero side). Data-only, fast, and **~3× more accurate** than the residual
  search at high noise on traces with a clear echo maximum. Falls back to
  `'residual'` when no concave echo peak is found (e.g. the trace already starts
  at the zero-time).

    On a **noisy** echo the vertex is the wrong statistic, so above a measured
    noise-to-amplitude ratio the estimator switches to the **centroid of the echo
    top**. Two things break the parabola together, and both get worse the broader
    $P(r)$ is: the echo top flattens, so its curvature — and with it the vertex
    $-b/2a$ — becomes a ratio of two quantities that are dominated by noise; and the
    argmax that anchors the fit window is a winner-take-all choice among many
    near-equal noisy samples, an error no local refinement can undo.

    The centroid needs no curvature. $V$ is even about $t_0$, so the centroid of any
    weight symmetric about $t_0$ **is** $t_0$; being *linear* in $V$ it averages the
    top samples instead of choosing between them, and it cannot diverge or fail to
    find a peak. Below the threshold nothing changes and the result is identical to
    the plain parabola, which is the usual case for well-averaged data — the switch
    is for traces where the per-point noise is a sizeable fraction of the echo
    amplitude.

    The gain shows up on **both** inversion engines, which is what distinguishes it
    from the accidental cancellation described under `xcheck`; it is largest for
    broad distributions at long distance, where the echo top is flattest.
- **`'residual'`** — minimize the V-space reconstruction residual. A candidate
  offset $s$ shifts both the time axis and the (data-anchored) background window,
  so only the kernel alignment changes; the residual is smooth with a single
  minimum, found by a coarse grid over the first `search_frac` of the trace plus a
  parabolic `refine`. For speed it uses a fixed-$\alpha$ *sequential* inversion on
  a capped distance grid; `**kwargs` pass through to [`deer_invert()`](#deer_invert)
  (`r`, `dim`, `fit_dim`, …). Robust when the echo maximum is ambiguous or absent.

**`xcheck`** is an opt-in cross-check, **off by default**: it computes the
`'residual'` estimate independently and, when it disagrees with the parabola by
more than `xcheck_tol_frac` of the trace span (~0.4 %), uses the residual instead.
It guards the parabola's one failure mode — a flat, shallow echo top at high noise,
where a late noise excursion can drag the vertex late — but does not improve the
recovered $P(r)$, so leave it off unless an accurate $t_0$ per se is the goal.

Returns $t_0$ in the same units as `t` (µs).

```python
t0 = deer.fit_zero_time(t, V, bg_start=1.0, r=r)
res = deer.deer_invert(t - t0, V, r=r, bg_start=1.0 - t0)
```

---

## deer_validate() { #deer_validate data-toc-label="deer_validate" }

```python
val = deer.deer_validate(t, V, r=None, bg_start=None, bg_starts=None,
                         bg_end=None, dim=3.0, fit_dim=False, alpha=None,
                         alpha_factor=1.0, reg_order=2, nu_dd=deer.NU_DD,
                         method='gcv', engine='sequential',
                         noise=0.0, n_noise=0, seed=0, percentiles=(5, 95))
```

**Validation by ensemble averaging**, in the style of the DeerAnalysis validation
tool. The regularization weight is selected **once** on the central trace (honouring
`alpha` / `alpha_factor`) and then held **fixed**, while the inversion is re-run
over a sweep of background-start times (and, optionally, added-noise
realizations). The ensemble of $P(r)$ is collapsed to a **median consensus curve**
plus a percentile **uncertainty band**.

A single GCV inversion of a noisy DEER trace leaves a spiky comb-like $P(r)$;
averaging across background choices suppresses those noise-driven spikes and yields
the smooth, banded distribution familiar from inter-laboratory ring tests
([Schiemann et al., *JACS* **2021**, 143, 17875](https://doi.org/10.1021/jacs.1c07371),
Fig. 4). Holding $\alpha$ fixed is both physically correct — validation probes
*background/noise sensitivity*, not the regularization choice — and what keeps it
fast (no per-trial L-curve scan).

!!! note "Mellin engine"
    $\alpha$ is not the Mellin regularizer, so for `engine='mellin'` the cutoff
    `tau_max` is pinned to the central trial instead (together with its `n_tau`
    grid and the split point `delta`). Without that, every trial would re-run the
    cutoff selection and the band would report the jump between cutoffs rather
    than the background sensitivity it claims to measure.

- **`bg_start`** — centre of the default background-start sweep (µs). `None` uses
  the trace midpoint.
- **`bg_starts`** — explicit sweep of background-start times. `None` builds a
  9-point grid spanning $\pm 7.5\%$ of the trace length around `bg_start`.
- **`alpha`, `alpha_factor`** — passed to [`deer_invert()`](#deer_invert) for the
  one-off $\alpha$ selection on the central trace; the result is then fixed.
- **`noise`, `n_noise`** — when both are positive, each background-start trial is
  repeated with `n_noise` Gaussian-noise realizations of standard deviation `noise`
  added to $V$ (estimate `noise` from the trace residual).
- **`engine`** — `'sequential'`, `'joint'`, `'mellin'`, `'gauss'`, `'none'`, or
  `'general'`, as in [`deer_invert()`](#deer_invert). Extra engine parameters
  (Mellin `delta` / `tau_max`, Gaussian `n_gauss` / `max_gauss`, the `bg_params`
  general-background coefficients, …) pass through via `**kwargs`.
- **`percentiles`** — the lower/upper percentiles of the band (default 5–95%).

Returns a dict:

| Key | Description |
| --- | ----------- |
| `r` | The distance axis |
| `P_density` | **Median** $P(r)$ density across the ensemble (the consensus curve — plot this) |
| `P_mean` | Ensemble-mean density (for reference) |
| `P_lower`, `P_upper` | The `percentiles` band (shade between these) |
| `ensemble` | All `n_trials` × `len(r)` trial densities |
| `n_trials` | Number of successful trials |
| `bg_starts` | The background-start grid that was swept |
| `alpha` | The fixed regularization weight |
| `peak`, `r_mean` | Peak position and first moment of the consensus curve |
| `base` | The single central inversion (its `form_factor` / `F_fit` / `background` / `l_curve`, for display) |
| `trials` | Per-trial `bg_start`, `r_mean`, `lambda`, `k` and `flagged` (whether that trial raised a background reliability flag) |
| `trial_spread` | `r_mean_spread`, `lambda_spread`, `n_flagged` / `n`, and `disagree` |

`disagree` is the one to act on: the reliability flags of a validated result
describe `base`, the central trial only, so a sweep in which the other trials land
on a different background solution would otherwise pass unnoticed. It is set when a
**majority** of trials raise a flag, or when the trial mean distances span more
than max(0.15 nm, 5%). A single flagged trial is not enough — on healthy data that
fires often while the answer is unaffected.

!!! warning "One row, one curve"
    `peak` / `r_mean` describe the **median** curve, while `base` carries the
    $\lambda$, $k$, $\alpha$, $R^2$ and reliability flags of the **central trial**.
    Do not mix them into one reported row: a pointwise median of several densities
    is not itself a density, and its width and skew can fall outside the range of
    every individual trial. Quote shape descriptors from one or the other, and say
    which.

```python
val = deer.deer_validate(t, V, r=r, bg_start=1.0, alpha_factor=2.0)
print(f"peak r = {val['peak']:.2f} nm  over {val['n_trials']} trials")
# plot the band:  fill_between(val['r'], val['P_lower'], val['P_upper'])
#         median:  plot(val['r'], val['P_density'])
```

In the Data Treatment GUI this is the **"Validate (background sweep → P(r) band)"**
checkbox; the distance view then shows the median curve over its shaded band.

---

## deer_invert_mellin() { #deer_invert_mellin data-toc-label="deer_invert_mellin" }

```python
res = deer.deer_invert_mellin(t, V, r=None, bg_start=None, bg_end=None,
                              dim=3.0, fit_dim=False, nu_dd=deer.NU_DD,
                              delta=None, tau_max=30.0, n_tau=601,
                              bg_engine='joint', n_mc=0, ci_z=1.96, seed=0,
                              taumax_method='penalty', noise_space='V',
                              wiener=0.0, taumax_extend=True,
                              extend_short_frac=0.18, fit_rmin_abs=2.0,
                              fit_rmin_width=0.5, signed_fit=True,
                              taper_short=True)
```

**Model-free** DEER inversion by the analytic integral **Mellin transform**
(Matveeva, Nekrasov & Maryasov, *PCCP* **2017**,
[10.1039/C7CP04059H](https://doi.org/10.1039/C7CP04059H)). No Tikhonov, no NNLS,
no L-curve: the distance distribution is recovered in closed form, so it is **not
broadened** and bimodal peaks are **not merged**. Also reachable as
`deer.deer_invert(..., engine='mellin')`.

Writing the (background-corrected, normalized) form factor as a multiplicative
convolution over the dipolar variable $w = 2\pi\nu_{dd}/r^3$,

$$
F(T) = \int_0^\infty p(w)\,\varphi(wT)\,dw,\qquad
\varphi(u) = \int_0^1 \cos\!\big(u(1-3x^2)\big)\,dx,
$$

the Mellin transform separates the variables: with $\tilde V(s)$, $\Phi(s)$, $P(s)$
the Mellin images of $F$, $\varphi$, $p$, one has $\tilde V(s) = P(1-s)\,\Phi(s)$,
so on the critical line $s = \tfrac12 + i\tau$ (using that $F$, $\varphi$ are real)

$$
P(\tfrac12 + i\tau) = \overline{\tilde V(\tfrac12+i\tau)\,/\,\Phi(\tfrac12+i\tau)},
$$

and the inverse Mellin transform gives $p(w)$ directly; the Jacobian maps it to
$f(r)$. The kernel image $\Phi$ is computed in closed form
([`mellin_kernel_spectrum()`](#mellin_kernel_spectrum)) and the signal image
$\tilde V$ by the $\delta$-split of the paper
([`mellin_signal_spectrum()`](#mellin_signal_spectrum)).

!!! info "The short-$r$ noise spike (\"double peak near $t=0$\")"
    The chain is linear, so noise enters $f(r)$ additively, and the $r$-space
    Jacobian ($\sim r^{-2.5}$) concentrates it into a spurious spike at short
    distances — the "double peak near $t=0$" in $P(r)$. The same high-$|\tau|$
    content also makes the forward-fit echo top decay too fast. Two mechanisms, both
    on by default, suppress this without distorting the real peaks:

    - **Noise-adaptive $\delta$.** $\delta$ splits the form factor into an analytic
      parabola term on $[0,\delta]$ and a numeric integral on $[\delta, T_\max]$; the
      noise enters through the numeric part. $\delta$'s floor and cap grow with the
      measured relative noise, so on noisy data more of the early signal goes to the
      clean analytic term. This fixes the spike at its source and does not touch the
      displayed density.
    - **`taper_short`** (default on). A geometric raised-cosine taper sends the
      reported $P(r)$ smoothly to zero at the unreliable short-$r$ edge. Its window
      is **absolute**: it ramps from the bottom of the grid up to at most
      `fit_rmin_abs` nm — the distance below which a DEER measurement is not
      meaningful anyway — over at most `fit_rmin_width` nm, and it switches off
      entirely on a grid that already starts above `fit_rmin_abs`. Because the
      window is fixed in nanometres rather than as a fraction of the grid, the
      reported distribution does not change when you widen or narrow the distance
      axis. A genuine short-$r$ peak is attenuated rather than deleted; the tapered
      density also feeds `F_fit`. Set `taper_short=False` for the raw signed
      density.

    !!! warning
        The taper multiplies the **reported** density, not only the fit curve, and
        the area is renormalized afterwards. On a distribution with real
        population below `fit_rmin_abs` it therefore shifts the reported mean
        distance and the relative peak areas: keep the distance grid starting at
        or above the shortest distance you intend to quote, or set
        `taper_short=False`.

    Together they remove the short-$r$ spurious mass and restore the echo-top width
    while $P(r)$ stays natural.

- **`bg_engine`** — `'joint'` (default), `'sequential'`, or `'none'`, how the form
  factor is prepared (see [`joint_background()`](#joint_background) /
  [`background_fit()`](#background_fit)). **This matters a lot:** the Mellin kernel
  $\varphi(wT)\to0$, so the recovered density cannot represent a DC pedestal left
  by an imperfect background — a too-shallow background shows up as a constant gap
  between data and fit. The joint engine gives a clean $F\to0$ and is the default.
  `'none'` sets $B(t)=1$ and fits only $\lambda$ — use it for data with **no**
  background (pre-corrected / simulated / full-modulation $\lambda\!\to\!1$): there
  the form factor decays to zero on its own, so fitting a background absorbs that
  dipolar decay and badly broadens $P(r)$.
- **`delta`** — the Mellin split point $\delta$ (µs): $[0,\delta]$ is integrated
  analytically, $[\delta, T_\max]$ numerically. The echo top is parabolic
  ($F\approx F_0 + b\,T^2$), so the analytic term keeps that quadratic and removes a
  systematic error in $F_\text{fit}$ at the echo (the "thin parabola" near $t=0$).
  `None` auto-selects $\delta$ where $F(\delta)\approx0.85$, then clips it to a
  noise-adaptive window (wider on noisier data). A larger $\delta$ moves the steep,
  noisy near-echo region into the clean analytic term, suppressing the short-$r$
  spike at its source. See [`mellin_signal_spectrum()`](#mellin_signal_spectrum) /
  [`mellin_delta()`](#mellin_delta).

    !!! warning "Shallow modulation + high noise"
        The form factor carries the electrical noise amplified by $1/\lambda$. Once
        that noise approaches the level drop the automatic $\delta$ tests for
        (roughly $\sigma/\lambda \gtrsim 0.09$), the split point and the echo-top
        curvature are both estimated on noise: they are therefore read off a lightly
        smoothed form factor in that regime, and the curvature is fitted over a wider
        window. Below that threshold nothing changes. Note the repair fixes the
        *forward fit*, which is otherwise held above the data across the whole echo
        top — it does not make $P(r)$ more accurate there, and at that noise level a
        Mellin distance distribution should not be quoted without a cross-check.
- **`tau_max`, `n_tau`** — the Mellin variable runs over $[-\tau_\max, \tau_\max]$
  with `n_tau` samples. The high-$\tau$ cutoff is the regularizer. **`tau_max=None`
  auto-selects it** by `taumax_method` (see below).
- **`taumax_method`** — how the auto cutoff is chosen. `'penalty'` (default)
  minimises the forward-fit RMS plus a penalty on the negative area of the signed
  density: the first term demands a good fit, the second stops the cutoff once it
  would only add high-$\tau$ noise. It self-adapts, keeping clean data sharp and
  noisy data smooth. `'discrepancy'` picks the smallest cutoff that reaches the noise
  floor, then applies the `taumax_extend` extension. `'lcurve'` is for comparison
  only — it under-regularizes on DEER (the residual is nearly flat in $\tau_\max$, so
  the corner is ill-defined).
- **`noise_space`** — `'V'` (default) or `'F'`: the space the noise floor and
  per-cutoff residual are measured in for the discrepancy selection. `'V'` (the
  whole background-normalized curve) is stationary and robust; `'F'` (the
  background-corrected form factor) is noise-amplified toward the tail.
- **`taumax_extend`** (default on) — a resolution-aware extension of the discrepancy
  cutoff. The discrepancy stops at the noise floor, but $P(r)$ can keep sharpening
  past it, so the cutoff is pushed up as long as the short-$r$ leakage (bottom
  `extend_short_frac` of the grid) keeps dropping, and stopped at the first increase.
  Clean data extends; noisy data stays put. Used only with
  `taumax_method='discrepancy'`; the default `'penalty'` method does not need it.
- **`taper_short`** (default `True`) — smoothly taper the reported $P(r)$ to zero at
  the unreliable short-$r$ edge with a geometric raised-cosine, removing the short-$r$
  noise spike while leaving the mid- and long-$r$ density unchanged. The tapered
  density also feeds `F_fit`. See the info box above. `False` returns the raw signed
  density.
- **`fit_rmin_abs`, `fit_rmin_width`** (nm) — the taper window: it ramps from the
  bottom of the distance grid up to at most `fit_rmin_abs`, over at most
  `fit_rmin_width`, and is switched off entirely when the grid starts above
  `fit_rmin_abs`. Absolute distances, so editing the distance axis does not change
  the reported distribution.
- **`signed_fit`** (default `True`) — score the automatic `tau_max` selection against
  the honest signed density rather than the clipped, tapered one. Set `False` for
  low-$\lambda$ data, where a short-$r$ negative spike can otherwise pull the
  selection. **With `taper_short=True` (the default) it does not change `F_fit`**,
  which is always built from the tapered density; it changes only which cutoff the
  automatic selection lands on, and so the result at the next run.
- **`wiener`** (default `0` = off) — strength of a Wiener-regularized inverse filter
  on the kernel-image division. The plain inverse $1/\Phi(\tau)$ amplifies noise at
  high $|\tau|$, which the $r$-space Jacobian concentrates into a short-$r$ spike.
  The filter rolls that off, with its $\varepsilon$ scaled by the measured tail noise
  so it does nothing on clean data. A value $\approx 0.12$ removes the spike at
  moderate noise. Rarely needed, since the adaptive $\delta$ and `taper_short`
  already handle the spike.
- **`n_mc`** — number of Monte-Carlo noise realizations for the confidence band
  (0 = off). The band is built by **additive-noise propagation**: the white
  electrical-noise level is read from the **decayed tail of $V$** by smoothing
  (returned as `noise_level`), added to the smooth $V$ fit, and propagated through
  the *fixed* background to $F$ — so $F$ inherits the realistic $1/(\lambda B)$
  amplification toward the tail. The band is the **per-distance STD** across the
  realizations: `P_lower`/`P_upper` $= $ `P_density` $\mp\,$`ci_z`$\cdot$`P_std`.
  ~100 realizations are typical.

!!! tip "Automatic cutoff — RMS penalized by symmetric noise (default)"
    The cutoff $\tau_\max$ regularizes the inversion: the forward-fit RMS falls as
    it captures the parabolic echo top, then sits on a **broad noise-floor
    plateau**, so neither chasing its minimum (over-extends, injects the noisy
    high-$\tau$ spectrum into $P(r)$) nor the discrepancy floor (under-shoots before
    $P(r)$ has sharpened) is right. The injected noise enters the area-normalized
    **signed** density as paired $+$bump/$-$dip excursions, so its $|$negative
    area$|$ (`neg`) measures it directly. The default `'penalty'` method picks
    $\operatorname{argmin}\big(\text{rmsF}/\min(\text{rmsF}) + \text{neg}\big)$: the
    ratio term ($\ge 1$, large while the echo top is under-resolved) forces an
    adequate fit, the `neg` term halts the extension the moment the cutoff would
    only add symmetric noise. Self-adapting: clean data plateaus late (sharp $P(r)$
    kept), noisy data accrues `neg` early (stays smooth). `whiteness` flags a residual
    that is still structured (see [`residual_whiteness()`](#residual_whiteness)).

!!! warning "Reading `sigma_fit` and `sigma_noise`"
    `sigma_noise` is the last 30% of the **same** residual `sigma_fit` is computed
    from, so the two are not independent and their ratio cannot separate over- from
    under-fitting: a ratio below 1 says only that the tail is fit *worse* than
    average, which usually points at the background. Compare `sigma_fit` with
    `noise_level` — the model-free noise level of $V$ — for a fit-quality verdict.
    Over-fitting in this engine is invisible to any residual statistic, because the
    injected structure is paired $\pm$ excursions in $P(r)$ that the forward kernel
    averages out; read `neg_area` instead, which grows monotonically with
    `tau_max`.

Returns the same dict shape as [`deer_invert()`](#deer_invert) (so the GUI and
exporters are shared), with these Mellin-specific keys:

| Key | Description |
| --- | ----------- |
| `engine` | `'mellin'` |
| `P_density` | Recovered **signed** density (area-normalized; short-$r$ noise ripples kept, can be < 0) — plot this |
| `P_signed_density` | Alias of `P_density` (kept for back-compat) |
| `P_lower`, `P_upper` | Monte-Carlo band $= $ `P_density` $\mp$ `ci_z`·`P_std` (when `n_mc > 0`; else `None`) |
| `P_std` | Per-distance STD across the MC realizations (when `n_mc > 0`) |
| `noise_level` | White electrical-noise σ read from the decayed tail of $V$; `NaN` when the trace is too short to measure it, `0.0` when the tail is exactly constant |
| `ci_kind` | `'mc_fixed_bg'` — the band conditions on the fitted background, λ, `tau_max` and `delta` |
| `ci_unavailable` | Why no band was produced although one was requested (empty otherwise) |
| `delta`, `tau_max` | The split point and cutoff used |
| `auto_taumax` | Whether `tau_max` was auto-selected |
| `sigma_fit`, `sigma_noise` | Forward-fit residual over $t>0$, and the last 30% of **that same residual** |
| `neg_area` | Negative area of the signed density — the over-fit indicator (grows with `tau_max`) |
| `whiteness` | Residual-whiteness goodness-of-fit dict (Durbin–Watson, lag-1 autocorrelation, ACF + white-noise band) — see [`residual_whiteness()`](#residual_whiteness) |
| `tau`, `V_image`, `kernel_image` | The $\tau$ grid and the Mellin spectra $\tilde V(\tau)$, $\Phi(\tau)$ |

`alpha` is `NaN` and `l_curve` is `None` (no Tikhonov regularization here).

```python
res = deer.deer_invert_mellin(t, V, r=r, bg_start=1.0,
                              tau_max=None, n_mc=50)   # auto cutoff + CI band
peak = res['r'][res['P_density'].argmax()]
print(f"peak r = {peak:.2f} nm, sigma_fit/sigma_noise = "
      f"{res['sigma_fit']/res['sigma_noise']:.2f}")
```

---

## deer_invert_gauss() { #deer_invert_gauss data-toc-label="deer_invert_gauss" }

```python
res = deer.deer_invert_gauss(t, V, r=None, bg_start=None, bg_end=None,
                             dim=3.0, fit_dim=False, nu_dd=deer.NU_DD,
                             n_gauss=None, max_gauss=4, bg_engine='joint',
                             bg_params=None, ic='aicc', n_mc=0, ci_z=1.96,
                             seed=0, sigma_min=None, sigma_max=None,
                             ci_mode='linear', ci_level=0.95,
                             prune_spurious=True, weight_min=0.02,
                             spike_weight_max=0.10, method='lsq',
                             mc_trials=30000, mc_tol=0.5)
```

**Parametric** DEER inversion: model $P(r)$ as a **sum of $N$ Gaussians** and fit
their centres, widths and amplitudes (the DeerAnalysis "Gaussian" mode / DeerLab
`dd_gaussN` approach). Also reachable as `deer.deer_invert(..., engine='gauss')`.
Complements the regularized ([`deer_invert()`](#deer_invert)) and model-free
([`deer_invert_mellin()`](#deer_invert_mellin)) engines: when the distribution
really is a few discrete modes this is the most robust, and — unlike a regularized
inversion — it gives genuine **parametric error bars** on each peak.

$$
P(r) = \sum_{k=1}^{N} a_k\,\exp\!\Big(\!-\tfrac{(r-r_k)^2}{2\sigma_k^2}\Big),
\qquad a_k,\ \sigma_k > 0 .
$$

The `'lsq'` solver fits the Gaussians, the background, and the modulation depth
$\lambda$ together, directly to $V(t)$ (DeerLab-style):

$$
V(t) = A\,\big[\,1 - S + (K\,\text{masses})(t)\,\big]\,B(t),
\qquad S = \textstyle\sum_k(\text{masses}) = \lambda .
$$

This is more robust than fitting a background first and dividing it out. On a
compact, multi-peak $P(r)$ the separate background step absorbs real signal, which
distorts the form factor and leaves a residual the fit then covers with a spurious
extra peak. Fitting everything at once avoids that, and an ideal $N$-Gaussian trace
is recovered exactly. $\lambda$ comes out as the total Gaussian mass $S$, and the
free amplitude $A$ absorbs the small echo-top scaling, so no extra scale parameter
is needed.

!!! note "Seeding and the long-distance width floor"
    Two safeguards keep the fit reliable:

    - **Multi-start.** The fit starts from two seeds — the peaks of a quick Tikhonov
      pass, and an even spread across the distance range — and keeps the better
      result. The Tikhonov peaks alone can land every component on the dominant peak,
      from where the fit never finds a weak long-distance mode; missing that mode
      leaves its slow oscillation in the residual.
    - **Width floor.** At long distances the dipolar frequency is low, so a finite
      trace length cannot resolve a narrow width. Left free, the fit collapses a
      weak long mode into a near-delta spike — a tall thin peak in $P(r)$ that adds
      an oscillation to the residual. Each component's width is therefore floored at
      the resolution limit for its distance,
      $\sigma_\text{res}(r)\approx r^4/(27\,\nu_{dd}\,T)$. Short, well-resolved peaks
      are unaffected. Set `sigma_min` to override.

- **`n_gauss`** — force a fixed number of components. `None` (default) selects $N$
  automatically (see `ic` / `prune_spurious`).
- **`max_gauss`** — largest $N$ tried during automatic selection (default 4).
- **`ic`** — information criterion for automatic $N$: `'aicc'` (default, corrected
  Akaike), `'aic'`, or `'bic'` (heavier penalty ⇒ fewer components).
- **`bg_engine`** — which background is co-fit with the Gaussians: `'joint'`
  (default, a stretched exponential $B(t)=e^{-(k|t|)^{d/3}}$, with $d$ floated only
  when `fit_dim=True`), `'none'` (no background, $B=1$ — for pre-corrected or
  full-modulation traces), or `'general'` (an empirical background shape held fixed
  while $\lambda$ and $P(r)$ are still co-fit; see
  [`background_general()`](#background_general)).
- **`bg_params`** — coefficients for the `'general'` background (see
  [`background_general()`](#background_general)).
- **`ci_mode`** — per-component error bars: `'linear'` (default) or `'support'`
  (see the confidence-interval box below). **`ci_level`** is the confidence for
  `'support'` (default 0.95).
- **`prune_spurious`** / **`weight_min`** / **`spike_weight_max`** — parsimony
  guard against over-fitting (default on; see the box below).
- **`n_mc`** — when > 0, a parametric confidence **band** on $P(r)$ by sampling the
  fit covariance `n_mc` times (cheap, no re-inversion); `P_lower`/`P_upper`
  $=$ `P_density` $\mp$ `ci_z`·STD. (Ignored when `method='mc'`.)
- **`method`** — `'lsq'` (default, gradient least-squares) or `'mc'` (Dzuba/Matveeva
  frequency-domain Monte-Carlo; see the box below). **`mc_trials`** sets the
  stochastic-multi-start budget and **`mc_tol`** the ensemble MSD tolerance.
- **`sigma_min`, `sigma_max`** — component-width bounds. Setting `sigma_min`
  overrides the automatic distance-resolution floor (above) with a flat lower bound
  $\max(2.5\,dr,\,\text{`sigma_min`})$ for every component. The upper bound defaults
  to half the distance range.

!!! info "Confidence intervals: `'linear'` vs `'support'`"
    `ci_mode='linear'` (default) reports the 1σ diagonal of the linearized
    covariance $(J^\top J)^{-1}\sigma^2$. It is fast, symmetric, and good for live
    use.

    `ci_mode='support'` computes rigorous support-plane / profile-likelihood
    intervals (Stein, Beth & Hustedt, *Methods Enzymol.* **2015**,
    [10.1016/bs.mie.2015.07.031](https://doi.org/10.1016/bs.mie.2015.07.031)). Each
    centre and $\sigma$ is fixed in turn while all other parameters are re-fit, and
    the interval is taken where the residual sum of squares rises past an F-test
    threshold. This accounts for parameter correlations and gives **asymmetric**
    intervals (`center_ci_lo/hi`, `sigma_ci_lo/hi`), which the linearized bar can
    misstate when the $\chi^2$ surface is not parabolic. It costs a fit per grid step
    (~1–5 s); opt-in.

!!! note "`method='mc'` — frequency-domain Monte-Carlo"
    An alternative solver after Dzuba, *JMR* **269** (2016) 1 and Matveeva *et al.*,
    *Z. Phys. Chem.* **231** (2017) 463. The Gaussian parameters are found by a random
    search in the dipolar frequency (Pake) domain instead of gradient descent in time.
    `mc_trials` random parameter sets are drawn, each locally polished, and the one
    whose Pake spectrum best matches the data is kept. This has two advantages over
    `'lsq'`: the random restarts cannot get stuck on a floor-width spike, and the
    frequency-domain comparison is naturally immune to ESEEM peaks and background
    error. The data-consistent trials form an ensemble; its per-$r$ percentiles give
    a confidence band (`P_lower`/`P_upper`) and its spread sets `center_err`/`sigma_err`.

    On clean synthetic data `'mc'` performs about the same as `'lsq'`, so it is
    opt-in (~seconds). Its real value is robustness to ESEEM and background artifacts
    on measured data, plus the honest ensemble error band. `n_mc` and `ci_mode` are
    ignored in this mode.

!!! tip "How $N$ is chosen (`prune_spurious`)"
    Every $N$ from 1 to `max_gauss` is fit, and the information criterion (`ic`)
    picks the best. DEER traces are heavily oversampled, so at low noise the
    criterion sometimes adds one extra Gaussian to absorb residual structure. Such a
    component is recognizable: it sits at the width floor and carries little weight
    ($<$ `spike_weight_max`), or it carries negligible weight ($<$ `weight_min`) at
    any width. With `prune_spurious` on (default) the chosen $N$ is the best fit that
    contains no such component, so a simple bimodal is not reported as 3–4 Gaussians.
    `n_gauss_ic` is the unpruned pick and `pruned` flags whether a reduction
    happened; forcing `n_gauss` bypasses it.

    Only weight, not width, condemns a component, because a real long-distance mode
    also sits near the width floor (the kernel constrains large-$r$ widths weakly).
    A floor-width peak with substantial weight is kept; only floor-width **and**
    low-weight is removed. With the joint fit, multi-start seeding, and the width
    floor doing the main work, pruning is now just a light backstop: a real 3–4
    Gaussian distribution is resolved, while a simple bimodal stays $N=2$.

Returns the same dict shape as [`deer_invert()`](#deer_invert) (shared GUI /
exporters), with these Gaussian-specific keys:

| Key | Description |
| --- | ----------- |
| `engine` | `'gauss'` |
| `components` | list of `{amplitude, center, sigma, weight, center_err, sigma_err}` per Gaussian (plus `center_ci_lo/hi`, `sigma_ci_lo/hi` when `ci_mode='support'`) |
| `n_gauss` | the chosen number of components |
| `n_gauss_ic`, `pruned` | the unpruned criterion pick, and whether pruning reduced $N$ |
| `aic`, `aicc`, `bic` | the information criteria of the chosen model |
| `ic`, `ci_mode`, `ci_level` | the criterion and CI mode used |
| `ic_curve` | list of `(N, criterion, rss)` over the $N$ tried |
| `P_lower`, `P_upper`, `P_std` | parametric band on the density (when `n_mc > 0`; else `None`) |
| `noise_level` | white electrical-noise σ from the decayed tail |

`alpha` is `NaN` and `l_curve` is `None` (no Tikhonov regularization here).

```python
res = deer.deer_invert_gauss(t, V, r=r, bg_start=1.0, ci_mode='support')
for c in res['components']:
    print(f"r = {c['center']:.3f} (-{c['center']-c['center_ci_lo']:.3f}"
          f"/+{c['center_ci_hi']-c['center']:.3f}) nm, "
          f"sigma = {c['sigma']:.3f} nm, weight = {c['weight']:.2f}")
print(f"N = {res['n_gauss']} (AICc pick {res['n_gauss_ic']}, pruned={res['pruned']})")
```

---

## joint_background() { #joint_background data-toc-label="joint_background" }

```python
bg = deer.joint_background(t, V, bg_start=None, bg_end=None, dim=3.0,
                           fit_dim=False, nu_dd=deer.NU_DD, n_r=60,
                           rate_alpha=1.0, lam_pin_frac=0.5)
```

The λ-pinned joint background, returning **only** the background (same dict shape
as [`background_fit()`](#background_fit)). The rate is fit on a coarse internal
distance grid (`n_r`) at a fixed regularization (`rate_alpha`): $k$ and $\lambda$
are insensitive to the $P(r)$ resolution, so this is ~30× faster than a full joint
inversion — fast enough to re-run per background-start during Mellin validation.
This is the **shared** background fit of **both** inversion engines:
[`deer_invert_joint()`](#deer_invert_joint) (Tikhonov, `engine='joint'`) and
[`deer_invert_mellin()`](#deer_invert_mellin) (`bg_engine='joint'`) both call it.

Two robustness measures — both critical to **either** engine (the truncated-grid
rate fit is exactly what stops a gentle background being absorbed as spurious
long-$r$ $P(r)$ mass, which broadens the Tikhonov $P(r)$ *and* leaves a pedestal
the Mellin kernel $\varphi\to0$ cannot represent):

- **λ pinned over the later, more-decayed part of the tail** (`lam_pin_frac`, the
  last 50 % of $[\text{bg\_start}, T_\max]$ by default). $\lambda$ is the
  *asymptotic* baseline ($F\to0$); pinning $\langle F\rangle = 0$ over the whole
  tail biases it high when a broad/long-$r$ component has not decayed ($\langle
  F\rangle > 0$ there), underestimating $\lambda$ and pushing $k$ too steep — a
  residual tail droop the Mellin engine cannot represent. The later window is more
  decayed, so $k$ returns near-true. (The rate-fit residual still spans the whole
  trace; `bg_end` only seeds the initial guess.)
- **Rate fit on the trace-supported distance cap** $r_\max \approx
  5\,(T_\max/2)^{1/3}$ nm, which keeps $k$ determined on short single-peak traces
  and stops a gentle background being absorbed as spurious long-$r$ mass. A second,
  wider cap used to be fitted alongside it and preferred unless it collapsed toward
  a flat background; that discrete choice was removed, because the objective is
  multi-modal in $\log k$ and the two branches could differ by 19× in $k$ while
  differing by 0.03% in the objective — so a background-start cursor moved by a
  single sampling step could flip the reported mean distance by ~1 nm.

Reliability keys in the returned dict — each also raises a `RuntimeWarning`, and the
Data Treatment / DEER windows render them as an orange flag line:

| Key | Meaning |
| --- | ------- |
| `lambda_raw`, `lambda_clamped` | The raw pinned modulation depth, and whether it hit the [0.02, 0.95] clamp |
| `tail_abs_F` | $\langle|F|\rangle$ over the pin window; above ~0.05 the tail has **not** decayed and $\lambda$ is a guess |
| `k_ref`, `k_ratio`, `k_disagrees` | The sequential tail-fit rate, and how far the joint rate sits from it |
| `k_at_bound` | $k$ landed on an edge of its search bracket — it carries no information there (this happens when the sequential seed itself collapses) |
| `rmax_cap` | The distance cap the rate fit used |

---

## mellin_kernel_spectrum() { #mellin_kernel_spectrum data-toc-label="mellin_kernel_spectrum" }

```python
Phi = deer.mellin_kernel_spectrum(tau, n_u=512)
```

Mellin image $\Phi(\tfrac12 + i\tau)$ of the orientation-averaged dipolar kernel
$\varphi(u) = \int_0^1\cos(u(1-3x^2))\,dx$, on the critical line, vectorized over
`tau`. Computed in closed form — $\Phi(s) = \Gamma(s)\cos(\pi s/2)\int_0^1
|1-3x^2|^{-s}dx$, the $x$-integral splitting (via $x = x_0\sin\theta$ and
$x = x_0\cosh u$, $x_0 = 1/\sqrt3$) into an exact Beta-function term plus a smooth
quadrature — avoiding ${}_3F_3$ hypergeometric. Valid for
$0 < \operatorname{Re} s < 3/2$.

---

## mellin_signal_spectrum() { #mellin_signal_spectrum data-toc-label="mellin_signal_spectrum" }

```python
Vimg = deer.mellin_signal_spectrum(t, F, tau, delta, F0=1.0, du=0.02,
                                   parabolic=True, fit_level=0.80)
```

Mellin image $\tilde V(\tfrac12 + i\tau)$ of the form factor $F(T)$ via the
$\delta$-split: on $[0,\delta]$ integrate analytically; on $[\delta, T_\max]$
substitute $u = \ln T$ so $e^{i\tau\ln T} \to e^{i\tau u}$ has a *constant*
frequency $\tau$, and integrate on a log-$T$ grid of step `du`
(choose $du \lesssim \pi/\max|\tau|$). `t` in µs, only $T>0$ used; $F$ normalized
to $F(0) = $ `F0`.

The echo top is **parabolic** ($F$ is even in $T$ with negative curvature), so with
**`parabolic`** (default) the $[0,\delta]$ term keeps the quadratic
$F\approx F_0 + b\,T^2$ instead of assuming $F$ constant:

$$
\int_0^\delta (F_0 + b\,T^2)\,T^{s-1}\,dT
  = F_0\,\frac{\delta^s}{s} + b\,\frac{\delta^{s+2}}{s+2},\qquad s=\tfrac12+i\tau.
$$

The curvature $b$ is least-squares fit over a widened low-$T$ window (out to where
$F$ has fallen to `fit_level`·$F_0$, and never narrower than **three** positive
samples, so a coarse step cannot silently drop the term back to the constant-$F$
split). This removes a systematic error in the recovered $F_\text{fit}$ at the echo
(the "thin parabola" near $t=0$) and lets $\delta$ be widened. Set
`parabolic=False` for the original constant-$F$ split.

---

## mellin_inverse() { #mellin_inverse data-toc-label="mellin_inverse" }

```python
p_w = deer.mellin_inverse(P_tau, tau, w)
```

Inverse Mellin transform on the line $s = \tfrac12 + i\tau$ back to $p(w)$:
$\operatorname{Re}[p(w)] = \tfrac{1}{2\pi}\,w^{-1/2}\int
\operatorname{Re}[P(\tau)\,e^{-i\tau\ln w}]\,d\tau$, with `P_tau` $= P(\tfrac12 +
i\tau)$ sampled on `tau`. Returns the real $p(w)$ for each `w`.

---

## mellin_delta() { #mellin_delta data-toc-label="mellin_delta" }

```python
delta = deer.mellin_delta(t, F, level=0.95, floor=0.09, cap=0.12, floor_ratio=2.0)
```

Practical Mellin split point $\delta$: the first $T>0$ where the form factor has
fallen to `level` of $F(0)$ ($F(\delta)\approx0.95$). Falls
back to the first positive sample if $F$ never drops that far.

The raw level estimate is then **clipped to `[floor, cap]`** (µs; set either to
`None` to disable, and both are clamped to the last sample). The **floor** widens a
too-narrow analytic parabolic $[0,\delta]$ echo-top anchor (the "thin parabola"),
which otherwise leaves the recovered $F_\text{fit}$ top too steep and the short-$r$
density unstable. The **cap** ($\approx 120$ ns) stops a slow-decaying (long-$r$)
trace from over-smoothing $P(r)$ by integrating too much of the modulation
analytically.

**`floor_ratio`** bounds how far the floor may stretch $\delta$ beyond the trace's
*own* decay scale: $\delta$ is raised to at most `floor_ratio` × the raw crossing.
Without it the floor is an absolute time, so for $r_0\lesssim2.5$ nm — where the raw
crossing is several times smaller than 90 ns — it hands most of the first dipolar
oscillation to a single parabola and the reconstruction collapses: at $r_0=1.6$ nm
the $P(r)$ overlap falls to **0.17**, against **0.68** with `floor_ratio=2`. Above
$\approx3$ nm the raw crossing already exceeds `floor/floor_ratio`, so the clamp
binds exactly as before.

---

## residual_whiteness() { #residual_whiteness data-toc-label="residual_whiteness" }

```python
w = deer.residual_whiteness(resid, max_lag=None)
```

**Residual-whiteness goodness-of-fit diagnostic** (DeerLab-style). An adequate DEER
fit leaves a **white** (uncorrelated) residual; a structured, *oscillating* residual
is the hallmark of a distance distribution that has not captured all the dipolar
modulation — typically an over-smoothed (too-broad) $P(r)$ at an over-regularized
cutoff, but also missing dipolar pathways or orientation selection. Such model
inadequacy shows up as **autocorrelation** even when the residual *amplitude* already
matches the noise level — so the discrepancy principle alone cannot see it (Edwards &
Stoll, *JMR* **288** (2018) 58; Fábregas Ibáñez *et al.*, *Magn. Reson.* **1** (2020)
209). Returned as a dict:

| Key | Description |
| --- | ----------- |
| `durbin_watson` | $\mathrm{DW}=\sum(e_i-e_{i-1})^2/\sum e_i^2\in[0,4]$; $\approx 2$ white, $<2$ positive autocorrelation (the oscillating-residual case), $>2$ anti-correlation |
| `acf1` | lag-1 autocorrelation $r_1=\sum e_i e_{i-1}/\sum e_i^2$ ($\approx 1-\mathrm{DW}/2$); 0 = white — the headline number |
| `acf`, `lags` | the autocorrelation function vs lag (for an autocorrelogram) |
| `ci95` | $\pm 1.96/\sqrt N$, the 95 % white-noise band for the ACF. This is the band for **raw** white noise; a fitted, regularized residual has a different null distribution and sits slightly anti-correlated, so `white` over-flags on that side |
| `offset` | $\mathrm{mean}(e)/\mathrm{std}(e)$, computed **before** the mean subtraction. DW and the ACF are taken on the demeaned residual, which is blind to a constant pedestal from a mis-fitted $\lambda$ or background — exactly the systematic this diagnostic is meant to catch. An $\lvert$`offset`$\rvert$ of order 1 is a bad fit however white it looks |
| `white` | bool, $\lvert r_1\rvert \le$ `ci95` (residual consistent with white noise) |

[`deer_invert_mellin()`](#deer_invert_mellin) runs it on the V-space fit residual and
returns it under `whiteness`. The standalone **DEER / PDS Analysis** tool surfaces it
as the *Residual* and *Residual ACF* top-plot views (autocorrelogram + white-noise
band) and the `DW`/$r_1$ verdict in the info panel, with `offset` shown alongside
when it exceeds $0.25\sigma$.

---

## distribution_moments() { #distribution_moments data-toc-label="distribution_moments" }

```python
d = deer.distribution_moments(r, P)
```

**Shape descriptors of a distance distribution** $P(r)$ — the quantities most PDS work
actually reports, computed from the non-central moments $M_n=\int r^n P(r)\,dr$ of the
clipped, area-normalized density (Nekrasov, Matveeva, Syryamina, Agarkin & Bowman,
*Phys. Chem. Chem. Phys.* **2026**, [DOI 10.1039/D5CP04144A](https://doi.org/10.1039/D5CP04144A),
Eqns. 6–7 & 17). Negative excursions (the signed Mellin output) are clipped before
normalizing so these stay proper distribution moments. Returns a dict:

| Key | Description |
| --- | ----------- |
| `mean` | mean distance $r_0=M_1$ (nm) |
| `width` | rms width $\delta r=\sqrt{M_2-M_1^2}$ (nm) |
| `skew` | skewness $\gamma=(M_3-3M_1\delta r^2-M_1^3)/\delta r^3$ |
| `m1`–`m4` | the raw non-central moments $M_1\ldots M_4$ |

Unlike a shape-overlap coefficient, the moments expose the *direction* of an error — a
shifted mean vs. a wrong width vs. a wrong skew.

---

## moment_error_apriori() { #moment_error_apriori data-toc-label="moment_error_apriori" }

```python
ME_n = deer.moment_error_apriori(eps, dt, n_points, n=1)
```

**A priori rms error of the $n$-th moment of $P(r)$ from random noise alone** — the
closed form of Nekrasov, Matveeva, Syryamina, Agarkin & Bowman, *PCCP* **2026**
([DOI 10.1039/D5CP04144A](https://doi.org/10.1039/D5CP04144A), Eqn. 9, uniform
acquisition):

$$ME_n=\frac{\varepsilon\,\Delta t^{\,s}}{I(s)}\sqrt{\tfrac14+\sum_{i=2}^{N_T-1} i^{\,2(s-1)}},\qquad s=\frac{n}{3}$$

with $I(s)=\Phi(s)/(2\pi\nu_\text{dd})^{s}$ the analytic dipolar-kernel integral
(their Eqns. 5–6: $I(1/3)=4.31512$, $I(2/3)=3.06158$, $I(1)=2.77339$,
$I(4/3)=2.56993$, matching $g_e=2.0023193$). Because the
Mellin transform is **additive**, the noise decouples from the (unknown) distribution,
so the precision of a moment is a property of the **acquisition** — it needs no
inversion and no ground truth.

- **`eps`** — per-point rms noise on the **normalized form factor** $F(t)$ ($F(0)=1$);
  for a background-corrected trace of modulation depth $\lambda$ this is the raw trace
  noise amplified by $1/(\lambda B)$, i.e. $\varepsilon\approx\sigma_\text{trace}/\lambda$.
- **`dt`** — time step **in nanoseconds** (the constants $I(s)$ are fixed with the
  dipolar frequency in GHz, i.e. time in ns); pass `dt_us*1e3`.
- **`n_points`** — number of dipolar-trace points ($t\ge0$).
- **`n`** — moment order (1–4). $n=1$ is the **mean distance**, the robust one: its
  $i^{-4/3}$ weight is dominated by the *early* points, so $ME_1$ is nearly flat in
  `n_points` — extending the trace does **not** improve the mean distance; lowering the
  early-point noise does (the paper's NUA$_1$ result).

Returns $ME_n$ in nm$^n$ (nm for the mean distance). Against the paper's reported
uniform-acquisition $\mathrm{std}(M_1)=0.0400$ nm for `eps=0.04, dt=24, n_points=231`
this gives $0.0411$ nm.

!!! warning "$ME_n$ is a noise floor, not a bound"
    It is the propagated **noise** error of the linear Mellin moment integral and
    nothing else — it carries no resolution and no regularization-bias term, so it
    is not a bound on the scatter of a recovered distance. In Monte-Carlo tests
    $\mathrm{std}(M_1)/ME_1$ reaches **2.6×**, and $\mathrm{RMSE}/ME_1$ — which also
    carries the bias — reaches $\approx40\times$, both on a trace too short to
    resolve the distance: $ME_1$ is smallest exactly where the answer is worst.
    Surfaced in the **DEER / PDS Analysis** tool as `mean r ± ME₁`.

---

## dipolar_kernel() { #dipolar_kernel data-toc-label="dipolar_kernel" }

```python
K = deer.dipolar_kernel(t, r, nu_dd=deer.NU_DD)
```

Orientation-averaged DEER kernel (no background, no modulation), shape
`(len(t), len(r))` with $K(0, r) = 1$. Evaluated in closed form via Fresnel
integrals:

$$
K(t, r) = \sqrt{\tfrac{\pi}{6a}}\,\big[\cos(a)\,C(z) + \sin(a)\,S(z)\big],\quad
a = \omega(r)\,|t|,\quad z = \sqrt{6a/\pi}.
$$

---

## dipolar_frequency() { #dipolar_frequency data-toc-label="dipolar_frequency" }

```python
nu = deer.dipolar_frequency(r, nu_dd=deer.NU_DD)
```

Perpendicular dipolar frequency $\nu_\perp(r) = \nu_{dd}/r^3$ [MHz], `r` in nm.

---

## background_fit() { #background_fit data-toc-label="background_fit" }

```python
bg = deer.background_fit(t, V, bg_start, bg_end=None, dim=3.0, fit_dim=False)
```

Fits the intermolecular background on the window $\text{bg\_start} \le t \le
\text{bg\_end}$ and returns the background-corrected form factor. $V$ is
normalized so $V(0) = 1$ — using a **quadratic-vertex estimate** of the echo top
(over ±5 samples around $t=0$) rather than the single nearest sample, so a noisy
echo-max point cannot scale the whole form factor and narrow the recovered echo
top at high noise. The tail window is fit to
$(1-\lambda)\,e^{-(k|t|)^{d/3}}$ with modulation depth $\lambda = 1 - A$, and

$$
B(t) = e^{-(k|t|)^{d/3}}, \qquad F(t) = \frac{V(t)/B(t) - (1-\lambda)}{\lambda}.
$$

Only the **fit window** is bounded by `[bg_start, bg_end]`; $B(t)$ and $F(t)$ are
still evaluated over the whole trace. `bg_end=None` uses everything past
`bg_start`.

Returns a dict with `lambda`, `k`, `dim`, `A`, `B`, `form_factor`, `V_norm`,
`t`, `bg_start`, `bg_end`, and the boolean `mask` of the fit window.

---

## background_general() { #background_general data-toc-label="background_general" }

```python
bg = deer.background_general(t, V, bg_start, bg_end=None,
                             a=None, b=None, c=None, d=None, fit=True)
```

A **flexible empirical** intermolecular background, an alternative to the
stretched-exponential [`background_fit()`](#background_fit) for traces whose
decay is not $e^{-(k|t|)^{d/3}}$. The tail baseline is modelled as

$$
g(t) = a\,\exp\!\big(b\,(t + c\,d^{\,t})\big),\qquad a,\ b,\ c,\ d\ \text{free},
$$

with $d^{\,t}$ a true power. The same convention as
[`background_fit()`](#background_fit): $V$ is normalized so $V(0)=1$, the tail
baseline $g(t)=(1-\lambda)\,B(t)$, so the background normalized to $B(0)=1$ is
$B(t)=g(t)/g(0)$ with $g(0)=a\,e^{\,bc}$ (since $d^{\,0}=1$), the modulation depth
$\lambda = 1 - g(0)$, and $F(t)=(V(t)/B(t)-(1-\lambda))/\lambda$. The amplitude
$a$ **cancels** in the shape $B(t)=\exp\!\big(b(t+c(d^{\,t}-1))\big)$ — which stays
strictly positive — so $b,c,d$ set the background shape and $a$ only its $t=0$
level (hence $\lambda$). Reachable as `deer.deer_invert(..., engine='general')`
and via `bg_engine='general'` in the Mellin / Gaussian engines.

- **`fit`** — when `True` (default), the four coefficients are fit on the tail
  window; any of `a`/`b`/`c`/`d` supplied are used as the initial guess. When
  `False`, they are used **directly** as the background (manual mode — no fitting).
- **`a`, `b`, `c`, `d`** — the coefficients (seeds when fitting, values when not).
  Time $t$ is in **microseconds**, so they act on $t$ in µs.

!!! note "Needs a clean (well-decayed) tail"
    With four free parameters the model has more freedom than a stretched
    exponential, so it will **chase residual dipolar modulation** if the fit window
    still contains it — on a clean, well-decayed tail it recovers $\lambda$ to ~1%,
    but on an under-decayed tail it over-estimates the modulation depth. When
    fitting, `d` is constrained so the $d^{\,t}$ term keeps $\ge 5\%$ of its $t=0$
    amplitude across the window ($d^{\,\text{span}}\ge 0.05$): otherwise $c$ and $d$
    are unconstrained by the tail (where $d^{\,t}$ has vanished) and the fit is
    degenerate. The model can still be over-parametrized for a gentle decay, so the
    fitted $a$ / $c$ may be large (a near-constant exponential trading against the
    offset) — mathematically valid, $\lambda$ is unaffected.

Returns the same dict shape as [`background_fit()`](#background_fit), with `k` /
`dim` $=$ `NaN` (not applicable) and the coefficients in `params`
(`a`, `b`, `c`, `d`) plus `model='general'`. In the **DEER / PDS Analysis** GUI
this is the **"General (a·exp[b(t+c·dᵗ)])"** background option, with an
**Auto (fit)** toggle and per-coefficient boxes — Auto fits and writes the fitted
values back, unchecking it uses the hand-set coefficients directly.

```python
# fit the four coefficients on the tail
bg = deer.background_general(t, V, bg_start=2.0)
print(bg['params'], 'lambda =', round(bg['lambda'], 3))

# or set them by hand (no fitting) and invert with that fixed background
res = deer.deer_invert(t, V, r=r, bg_start=2.0, engine='general',
                       bg_params=dict(a=0.6, b=-0.05, c=1.0, d=0.5, fit=False))
```

---

## tikhonov_nnls() { #tikhonov_nnls data-toc-label="tikhonov_nnls" }

```python
P = deer.tikhonov_nnls(K, F, alpha, L=None)
```

Non-negative Tikhonov solution of $K P = F$: minimizes
$\lVert K P - F \rVert^2 + \alpha^2 \lVert L P \rVert^2$ subject to $P \ge 0$ by
solving the augmented NNLS problem $[\,K;\ \alpha L\,]\,P = [\,F;\ 0\,]$. `L`
defaults to the 2nd-derivative operator from
[`regularization_matrix()`](#regularization_matrix). Returns the masses
$P \ge 0$.

---

## regularization_matrix() { #regularization_matrix data-toc-label="regularization_matrix" }

```python
L = deer.regularization_matrix(n, order=2)
```

Discrete derivative operator $L$ for Tikhonov smoothing. `order=0` → identity,
`1` → first difference, `2` → second difference (curvature, the default).

---

## l_curve() { #l_curve data-toc-label="l_curve" }

```python
lc = deer.l_curve(K, F, alphas, L=None, method='gcv')
```

Regularization scan over `alphas`: for each one solves the NNLS-Tikhonov problem
and records the residual norm `rho`, the roughness norm `eta`, the Menger L-curve
`curvature`, and the `gcv` score. The optimum is chosen by `method`:

- **`'gcv'`** (default) — minimum of the generalized cross-validation score.
  Matches DeerLab's `gcv` selection exactly on the same grid. GCV uses the
  (unconstrained) Tikhonov influence-matrix trace as the effective degrees of
  freedom paired with the NNLS residual; that approximation biases $\alpha$
  *upward* relative to a constrained-dof GCV, never downward.
- **`'curvature'`** — classic maximum-Menger-curvature L-corner. Unreliable on
  DEER: the L-curve is nearly *vertical*, so there is no well-defined corner and
  the pick lands at either end of the grid depending on the background window
  (see the tip at the top of this page). The search covers the interior points
  only — the first and last curvature entries are undefined — and falls back to
  GCV, with `corner_ok=False`, when no interior corner exists.

Returns a dict with `alphas`, `rho` (residual norms), `eta` (solution norms),
`curvature`, `gcv`, `alpha_opt`, `index`, `method`, `P` (the solution at the
chosen $\alpha$), `at_bound` and `corner_ok`.

`at_bound` is `True` when the pick sits on the first or last grid point — a
**clipped** value, not an interior optimum, meaning the true optimum lies outside
`alphas`; a `RuntimeWarning` is raised and the DEER window flags it next to the
background line. This is reachable at default settings on broad distributions,
where GCV wants $\alpha \approx 4\times10^3$ against the grid ceiling of $10^3$.

!!! note "$\alpha$ is in grid-dependent units"
    [`regularization_matrix()`](#regularization_matrix) returns the raw $[1,-2,1]$
    stencil, i.e. $\Delta r^2$ times the true second derivative, so a given numeric
    $\alpha$ means *more* smoothing on a coarser distance grid, and changing
    **Distance points** changes what $\alpha$ means. It is also not directly
    comparable to DeerLab's ($\alpha_{\text{here}}\,\Delta r^2 \approx
    \alpha_{\text{DL}}$).

---

## default_r_axis() { #default_r_axis data-toc-label="default_r_axis" }

```python
r = deer.default_r_axis(rmin=1.5, rmax=8.0, n=200)
```

Returns a linear distance grid (nm).

---

## simulate() { #simulate data-toc-label="simulate" }

```python
V = deer.simulate(t, r, P, lam=0.3, k=0.05, dim=3.0,
                  nu_dd=deer.NU_DD, noise=0.0, seed=None)
```

Forward-simulates a DEER trace from a distance distribution $P(r)$:

$$
V(t) = \big[(1-\lambda) + \lambda\,(K P_\text{masses})\big]\,e^{-(k|t|)^{d/3}}
\ (+\ \text{Gaussian noise}).
$$

`t` in µs, `r` in nm; returns $V(t)$ with $V(0) = 1$ (noise aside). Handy for
validating the inversion round-trip and for tests.

---

## NU_DD { #nu_dd data-toc-label="NU_DD" }

Module constant — the perpendicular dipolar frequency constant
$\nu_{dd} = 52.04\ \text{MHz·nm}^3$ (for $g = 2.0023$), so that
$\nu_\perp(r) = \nu_{dd}/r^3$. Override it via the `nu_dd` argument of the
kernel / simulate / invert functions for other $g$-values.

---

## References

The methods implemented here draw on the following papers:

1. A. G. Matveeva, V. M. Nekrasov, A. G. Maryasov, *Analytical solution of the
   PELDOR inverse problem using the integral Mellin transform.* **Phys. Chem.
   Chem. Phys.** *2017*, 19, 32381.
   [10.1039/C7CP04059H](https://doi.org/10.1039/C7CP04059H) — the analytic
   Mellin-transform inversion ([`deer_invert_mellin()`](#deer_invert_mellin)).
2. M. M. Nekrasov, A. G. Matveeva, S. A. Syryamina, D. V. Agarkin, M. K. Bowman,
   **Phys. Chem. Chem. Phys.** *2026*.
   [10.1039/D5CP04144A](https://doi.org/10.1039/D5CP04144A) — distance-distribution
   moments and a priori moment-error bars
   ([`distribution_moments()`](#distribution_moments),
   [`moment_error_apriori()`](#moment_error_apriori)).
3. O. Schiemann *et al.*, *Benchmark Test and Guidelines for DEER/PELDOR
   Experiments on Nitroxide-Labeled Biomolecules.* **J. Am. Chem. Soc.** *2021*,
   143, 17875. [10.1021/jacs.1c07371](https://doi.org/10.1021/jacs.1c07371) —
   distance-reliability ranges and validation guidelines.
4. R. A. Stein, A. H. Beth, E. J. Hustedt, *A Straightforward Approach to the
   Analysis of Double Electron–Electron Resonance Data.* **Methods Enzymol.**
   *2015*, 563, 531.
   [10.1016/bs.mie.2015.07.031](https://doi.org/10.1016/bs.mie.2015.07.031) —
   parametric Gaussian model and confidence intervals.
5. S. A. Dzuba, **J. Magn. Reson.** *2016*, 269, 1.
   [10.1016/j.jmr.2016.06.001](https://doi.org/10.1016/j.jmr.2016.06.001) —
   Monte-Carlo (Pake-doublet) solver for the Gaussian engine (`method='mc'`).
6. A. G. Matveeva *et al.*, **Z. Phys. Chem.** *2017*, 231, 463.
   [10.1515/zpch-2016-0830](https://doi.org/10.1515/zpch-2016-0830) — Monte-Carlo
   Gaussian inversion.
7. T. H. Edwards, S. Stoll, **J. Magn. Reson.** *2018*, 288, 58 — optimal Tikhonov
   regularization and residual-whiteness diagnostics
   ([`residual_whiteness()`](#residual_whiteness)).
8. L. Fábregas Ibáñez, G. Jeschke, S. Stoll, *DeerLab: a comprehensive software
   package for analyzing dipolar EPR spectroscopy data.* **Magn. Reson.** *2020*,
   1, 209. [10.5194/mr-1-209-2020](https://doi.org/10.5194/mr-1-209-2020) —
   reference implementation for cross-validation (covariance CI, GCV).
9. G. Jeschke *et al.*, *DeerAnalysis2006 — a comprehensive software package for
   analyzing pulsed ELDOR data.* **Appl. Magn. Reson.** *2006*, 30, 473.
   [10.1007/BF03166213](https://doi.org/10.1007/BF03166213) — the zero-time
   parabola fit, distance-reliability ranges and validation approach this module
   follows.
