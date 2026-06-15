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
(ill-posed). This module solves it two ways:

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

The dipolar **zero-time** can be fit automatically with
[`fit_zero_time()`](#fit_zero_time) before either inversion.

!!! tip "Why GCV is the default"
    A DEER L-curve is nearly *vertical* — the residual stays at the noise floor
    across decades of $\alpha$ — so the Menger-curvature "corner" is ill-defined
    and tends to latch onto a tiny $\alpha$, producing a spiky comb-like $P(r)$.
    GCV has a genuine minimum and picks a stable $\alpha$, so it is used by
    default. Switch to the L-corner with `method='curvature'` if you want it.
    GCV still tends to *under*-regularize on real noisy traces; nudge it heavier
    with `alpha_factor=2..4`, or use [`deer_validate()`](#deer_validate) to average
    over background choices for a smooth consensus $P(r)$ with an uncertainty band.

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
  [`background_fit()`](#background_fit). (With `engine='joint'` these only seed
  the initial background guess — the joint fit uses the whole trace.)
- **`dim`, `fit_dim`** — fractal background dimension (3 = homogeneous 3D); set
  `fit_dim=True` to float it.
- **`alpha`** — regularization weight. `None` selects it automatically by `method`.
- **`alpha_factor`** — multiplier applied to the *auto-selected* $\alpha$ (ignored
  when an explicit `alpha` is given). GCV (and AIC) tend to under-regularize the
  near-vertical DEER L-curve, leaving noise spikes in $P(r)$; a factor of 2–4
  reproduces the heavier hand-picked L-corner regularization used to obtain smooth
  distributions in inter-laboratory ring tests
  ([Schiemann et al., *JACS* **2021**, 143, 17875](https://doi.org/10.1021/jacs.1c07371)).
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
  transform — see [`deer_invert_mellin()`](#deer_invert_mellin)), or `'none'`
  (**no background**: $B(t)=1$, fit only the modulation depth $\lambda$ — for
  pre-corrected / simulated / full-modulation $\lambda\!\to\!1$ data; fitting a
  decay there would absorb the dipolar decay and badly broaden $P(r)$).
- **`**kwargs`** — forwarded to [`deer_invert_mellin()`](#deer_invert_mellin) when
  `engine='mellin'` (`delta`, `tau_max`, `n_tau`, `bg_engine`, `n_mc`, …);
  ignored otherwise.

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
| `P_lower`, `P_upper` | 95% covariance confidence band on the density (see [`tikhonov_ci()`](#tikhonov_ci)) |
| `kernel` | The dipolar kernel matrix $K$ |
| `alpha` | The regularization weight used |
| `l_curve` | The [`l_curve()`](#l_curve) result dict (or `None`) |
| `background` | The [`background_fit()`](#background_fit) result dict |
| `lambda`, `k`, `dim` | Modulation depth, background decay rate, dimension |
| `engine` | `'sequential'`, `'joint'`, or `'mellin'` |

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

the background decay rate $k$ (and $d$ when `fit_dim=True`) is the only nonlinear
unknown. For each trial rate the modulation depth $\lambda$ is **pinned** to the
mean baseline of $V/B$ over the background window $[\text{bg\_start},
\text{bg\_end}]$ — where the intramolecular form factor has decayed and
$V \approx (1-\lambda)\,B$ — and the non-negative regularized $P(r)$ follows from
$K P = (V/B - (1-\lambda))/\lambda$. The rate is chosen to minimize the
whole-trace residual $\lVert V - B[(1-\lambda) + \lambda K P]\rVert$ with
`scipy.optimize.minimize_scalar` (or `least_squares` when `fit_dim=True`).

!!! note "Why $\lambda$ is pinned to the baseline"
    With $\lambda$ left free the fit is **degenerate**: a near-flat background
    plus extra long-$r$ $P(r)$ mass reproduces $V$ just as well as the correct
    deeper background, and the rate collapses to $k \to 0$. Pinning $\lambda$ to
    the decayed-tail baseline removes that direction and recovers the correct
    background — matching DeerLab's joint fit. So here `bg_start`/`bg_end` set the
    **baseline window** (not just an initial guess).

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

Covariance-based confidence band on the regularized $P(r)$ — the asymptotic
(curvature) CI DeerLab shows by default, returned with every Tikhonov inversion.
For the linear Tikhonov estimator $P = (K^\top K + \alpha^2 L^\top L)^{-1} K^\top F$
the form-factor noise propagates as

$$
\operatorname{cov}(P) = \sigma^2\, M M^\top,\qquad
M = (K^\top K + \alpha^2 L^\top L)^{-1} K^\top,
$$

with $\sigma^2$ estimated from the fit residuals (effective dof
$= N - \operatorname{tr}(K M)$). Returns `(lower, upper)` at confidence `z`
(default 1.96 ≈ 95%) on the same density scale as `P/sum(P)/dr`, clipped at 0. The
non-negativity constraint is **not** propagated, so the band is a slightly
conservative linear approximation (as in DeerLab's moment-based CI).

---

## fit_zero_time() { #fit_zero_time data-toc-label="fit_zero_time" }

```python
t0 = deer.fit_zero_time(t, V, bg_start=None, bg_end=None,
                        n_grid=16, search_frac=0.15, refine=True,
                        method='parabola', drop=0.15, smooth_w=5,
                        xcheck=True, xcheck_tol_frac=0.004, **kwargs)
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
- **`'residual'`** — minimize the V-space reconstruction residual. A candidate
  offset $s$ shifts both the time axis and the (data-anchored) background window,
  so only the kernel alignment changes; the residual is smooth with a single
  minimum, found by a coarse grid over the first `search_frac` of the trace plus a
  parabolic `refine`. For speed it uses a fixed-$\alpha$ *sequential* inversion on
  a capped distance grid; `**kwargs` pass through to [`deer_invert()`](#deer_invert)
  (`r`, `dim`, `fit_dim`, …). Robust when the echo maximum is ambiguous or absent.

**`xcheck`** (default **off**, opt-in) targets the parabola's one failure mode: a
**flat, shallow echo top at high noise**. There the maximum is ill-defined and an
upward noise excursion late on the top drags the vertex tens of ns **late** — a
systematic late bias that grows with noise (≈ +27 ns at σ = 0.04 on the synthetic
benchmark, vs ~1 ns at low noise). With `xcheck` the `'residual'` estimate is
computed independently and, when the two disagree by more than `xcheck_tol_frac`
of the trace span (~0.4 %), the more robust residual is used.

!!! warning "Off by default — does not improve end-to-end accuracy"
    The cross-check lowers the **mean** $t_0$ error (5.1 → 4.0 ns on the
    benchmark) but is left off because it does **not** improve the recovered
    $P(r)$: (1) at extreme noise (σ = 0.04) the residual fallback is itself
    high-variance and can overshoot tens of ns *early*, raising the **worst-case**
    $t_0$ error (29.6 → 45.9 ns); (2) the Mellin forward model carries a small
    residual bias that a slightly-late $t_0$ happens to compensate, so a *more
    accurate* $t_0$ can **lower** the distance overlap (benchmark mean
    0.853 → 0.838). It helps moderate-noise traces (σ ≈ 0.02) but hurts the
    cleanest and the noisiest. Enable only when an accurate $t_0$ per se is the
    goal, not better $P(r)$.

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

- **`bg_start`** — centre of the default background-start sweep (µs). `None` uses
  the trace midpoint.
- **`bg_starts`** — explicit sweep of background-start times. `None` builds a
  9-point grid spanning $\pm 7.5\%$ of the trace length around `bg_start`.
- **`alpha`, `alpha_factor`** — passed to [`deer_invert()`](#deer_invert) for the
  one-off $\alpha$ selection on the central trace; the result is then fixed.
- **`noise`, `n_noise`** — when both are positive, each background-start trial is
  repeated with `n_noise` Gaussian-noise realizations of standard deviation `noise`
  added to $V$ (estimate `noise` from the trace residual).
- **`engine`** — `'sequential'`, `'joint'`, or `'mellin'`, as in
  [`deer_invert()`](#deer_invert). Extra Mellin parameters (`delta`, `tau_max`, …)
  pass through via `**kwargs`.
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
                              taumax_method='discrepancy', noise_space='V',
                              wiener=0.0, taumax_extend=True,
                              extend_short_frac=0.18)
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

!!! info "Noise groups at short $r$ — and is kept signed"
    The whole chain is linear, so noise enters $f(r)$ **additively**. It
    consistently piles up as ripples at **short distances**, below the reliable
    range — that is the method's signature, not real structure. Unlike Tikhonov it
    does not smear it across all $r$. The recovered `P_density` is kept **signed**
    (those ripples can dip below zero — they are *not* clipped/"corrected"). The
    forward fit `F_fit`, however, is built from the **non-negative** density: a
    negative density propagated through $K$ would flip the curvature of $F_\text{fit}$
    at $t=0$ into a spurious double peak, so the time-domain *model* is clipped
    (this changes only the fit curve, never the reported `P_density`, so $F_\text{fit}
    \ne K\,P_\text{density}$ exactly). The clipping concentrates the short-$r$ noise
    mass, so the echo top of $F_\text{fit}$ decays a little too fast at high noise —
    a clean no-double-peak fit is preferred over matching that last bit of the top.

- **`bg_engine`** — `'joint'` (default), `'sequential'`, or `'none'`, how the form
  factor is prepared (see [`joint_background()`](#joint_background) /
  [`background_fit()`](#background_fit)). **This matters a lot:** the Mellin kernel
  $\varphi(wT)\to0$, so the recovered density cannot represent a DC pedestal left
  by an imperfect background — a too-shallow background shows up as a constant gap
  between data and fit. The joint engine gives a clean $F\to0$ and is the default.
  `'none'` sets $B(t)=1$ and fits only $\lambda$ — use it for data with **no**
  background (pre-corrected / simulated / full-modulation $\lambda\!\to\!1$): there
  the form factor decays to zero on its own, so fitting a background absorbs that
  dipolar decay and badly broadens $P(r)$ (a $\sigma\,0.20$ Gaussian came out at
  $\sigma\,0.7$, overlap $0.81$ vs $0.98$ with `'none'`).
- **`delta`** — the Mellin split point $\delta$ (µs): on $[0,\delta]$ the form
  factor is integrated analytically, $[\delta, T_\max]$ numerically. The echo top
  is **parabolic** $F\approx F_0 + b\,T^2$, so the $[0,\delta]$ term keeps that
  quadratic (not constant $F$) — removing a systematic error in $F_\text{fit}$ right
  at the echo (the "thin parabola" near $t=0$) and letting $\delta$ be wider. `None`
  auto-selects $\delta$ at $F(\delta)\approx0.85$, then **clips it to $[90, 120]$ ns**:
  the floor keeps the analytic echo-top anchor wide enough on *sharp* (fast-decaying)
  distributions — which otherwise cross the level within a couple of samples and leave
  $F_\text{fit}$ too steep at $t=0$ — and the cap stops a slow-decaying long-$r$ trace
  from over-smoothing $P(r)$. (See
  [`mellin_signal_spectrum()`](#mellin_signal_spectrum) / [`mellin_delta()`](#mellin_delta).)
- **`tau_max`, `n_tau`** — the Mellin variable runs over $[-\tau_\max, \tau_\max]$
  with `n_tau` samples. The high-$\tau$ cutoff is the regularizer. **`tau_max=None`
  auto-selects it** by `taumax_method` (see below).
- **`taumax_method`** — `'discrepancy'` (default, noise-floor anchored) or
  `'lcurve'` (corner of $\log\sigma_\text{fit}$ vs $\log\lVert L_2 P\rVert$).
  L-curve is provided for comparison but under-regularizes on DEER (the residual
  is nearly flat in $\tau_\max$, so the corner is ill-defined) — prefer the default.
- **`noise_space`** — `'V'` (default) or `'F'`: the space the noise floor and
  per-cutoff residual are measured in for the discrepancy selection. `'V'` (the
  whole background-normalized curve) is stationary and robust; `'F'` (the
  background-corrected form factor) is noise-amplified toward the tail.
- **`taumax_extend`** (default on) — **resolution-aware extension** of the auto
  cutoff. The discrepancy stops once the data residual hits the noise floor, but
  $P(r)$ can keep **sharpening** past that point; the cutoff is then pushed up
  while the spurious **short-$r$ leakage** (bottom `extend_short_frac` of the $r$
  grid) keeps dropping, and stopped at the first increase. Self-adapting:
  clean/low-noise data extends (sharper echo top, bimodals resolved — a clean
  narrow Gaussian goes 0.92 → 0.96 overlap), noisy data stays at the discrepancy
  pick (leakage rises immediately). Only for `taumax_method='discrepancy'` with
  auto `tau_max`.
- **`wiener`** (default `0` = off) — strength of a **Wiener-regularized inverse
  filter** on the kernel-image division. The plain inverse $1/\Phi(\tau)$ amplifies
  noise where $\Phi$ is small (high $|\tau|$), and the $r$-space Jacobian
  ($\sim r^{-2.5}$) concentrates it into a spurious **short-$r$ spike** that can
  steal the real peak on noisy bimodal traces. The filter
  $\overline{\Phi}/(|\Phi|^2+\varepsilon)$ rolls that off, with $\varepsilon$ scaled
  by the measured tail noise so it is a no-op on clean data and leaves genuine
  short-$r$ peaks intact. A value $\approx 0.12$ helps at **moderate** noise
  ($\sigma\approx0.02$: removes the spike, overlap +0.1–0.2). Off by default — at
  extreme noise the result is dominated by zero-time/$\tau_\max$ auto-selection
  instability, where the filter is a net wash; enable it when the data are
  moderately noisy and show the tell-tale short-$r$ spike.
- **`n_mc`** — number of Monte-Carlo noise realizations for the confidence band
  (0 = off). The band is built by **additive-noise propagation**: the white
  electrical-noise level is read from the **decayed tail of $V$** by smoothing
  (returned as `noise_level`), added to the smooth $V$ fit, and propagated through
  the *fixed* background to $F$ — so $F$ inherits the realistic $1/(\lambda B)$
  amplification toward the tail. The band is the **per-distance STD** across the
  realizations: `P_lower`/`P_upper` $= $ `P_density` $\mp\,$`ci_z`$\cdot$`P_std`.
  ~100 realizations are typical.

!!! tip "Automatic cutoff — discrepancy anchored to the noise floor"
    The cutoff $\tau_\max$ regularizes the inversion: $\sigma_\text{fit}$ (the
    V-space forward residual) falls with $\tau_\max$ and **flattens** at the noise
    level. Chasing its *minimum* overshoots below the floor — an over-fit that
    injects the noisy high-$\tau$ spectrum into $P(r)$ (roughness explodes for a
    noise-level $\sigma_\text{fit}$ gain). So the routine instead picks the
    **smallest cutoff whose $\sigma_\text{fit}$ reaches the noise floor** within its
    statistical spread (floor from the V residual's successive differences
    $/\sqrt2$; tolerance $\sim 2/\sqrt{2N}$). This self-adapts: clean data needs a
    large cutoff to reach the tiny floor (keeps sharp features), noisy/short data
    reaches it early and stays smooth. `sigma_fit` and the tail `sigma_noise` are
    reported so the regime stays visible ($\approx$ matched, $\gg$ underfit,
    $\ll$ overfit).

Returns the same dict shape as [`deer_invert()`](#deer_invert) (so the GUI and
exporters are shared), with these Mellin-specific keys:

| Key | Description |
| --- | ----------- |
| `engine` | `'mellin'` |
| `P_density` | Recovered **signed** density (area-normalized; short-$r$ noise ripples kept, can be < 0) — plot this |
| `P_signed_density` | Alias of `P_density` (kept for back-compat) |
| `P_lower`, `P_upper` | Monte-Carlo band $= $ `P_density` $\mp$ `ci_z`·`P_std` (when `n_mc > 0`; else `None`) |
| `P_std` | Per-distance STD across the MC realizations (when `n_mc > 0`) |
| `noise_level` | White electrical-noise σ read from the decayed tail of $V$ |
| `delta`, `tau_max` | The split point and cutoff used |
| `auto_taumax` | Whether `tau_max` was auto-selected |
| `sigma_fit`, `sigma_noise` | Forward-fit residual vs tail noise floor (the discrepancy diagnostic) |
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

## deer_mellin_consensus() { #deer_mellin_consensus data-toc-label="deer_mellin_consensus" }

```python
res = deer.deer_mellin_consensus(t, V, r=None, bg_start=None, bg_end=None,
                                 dim=3.0, fit_dim=False, nu_dd=deer.NU_DD,
                                 n_t0=9, n_mc=6, chi_tol=1.05,
                                 taumax_window=(0.8, 0.9, 1.0, 1.1, 1.25),
                                 n_bg=5, bg_span_frac=0.05,
                                 gate_rel_noise=0.06, seed=0,
                                 percentiles=(2.5, 97.5), **kwargs)
```

**Robust ("consensus") Mellin inversion for noisy traces.** At high *relative*
noise ($\sigma/\lambda \gtrsim 0.06$) a DEER trace **does not determine** the
zero-time $t_0$ *or* the cutoff $\tau_\max$ — the V-space forward residual is white
(structureless) across a wide range of both, so they are **unidentifiable** and a
single auto-pick swings wildly per noise realization (recovered overlap can move
0.5→0.85 with $t_0$ changes the residual cannot even see). The honest answer is not
a sharper point estimate but a **consensus** over everything the data cannot rule
out, plus a **band** that shows the real uncertainty — the model-free analogue of
[`deer_validate()`](#deer_validate).

- **Identifiable** (rel noise `< gate_rel_noise`): the single pick is reliable, so
  it is returned with an `n_mc` Monte-Carlo band (`consensus=False`). Genuinely
  sharp clean distributions are left untouched.
- **Unidentifiable** (rel noise `≥ gate_rel_noise`): build an ensemble over the
  data-consistent zero-times (`n_t0` grid kept within `chi_tol` of the best
  forward-fit χ) × a `taumax_window` of cutoffs *around* the auto-pick (fractions,
  so it tracks the data), **plus an `n_bg`-point background-start sweep** (±
  `bg_span_frac` of the trace — subsuming the [`deer_validate()`](#deer_validate)
  axis) and `n_mc` measurement-noise realizations. The contributions are *additive*
  (not a full multiplied grid) to keep the cost down and the per-distance median
  smooth.
  The reported $P(r)$ is the ensemble **median** with a `percentiles` band
  (`consensus=True`).

Same return dict as [`deer_invert_mellin()`](#deer_invert_mellin) (so the GUI and
exporters are shared), plus `consensus` (bool), `rel_noise`, `n_trials`,
`t0`, `t0_consistent`, `tau_maxs`, `ensemble`, and `base` (the central single-pick
result, for the time-domain forward-fit display). `**kwargs` pass through to
[`deer_invert_mellin()`](#deer_invert_mellin) (`delta`, `taumax_method`, `wiener`, …).

!!! tip "When to use"
    On a synthetic benchmark this lifts the mean true↔recovered overlap on the
    hard, moderately-noisy cases by **+0.1 to +0.16** (e.g. a narrow + broad
    bimodal at $\sigma=0.02$: 0.68 → 0.84) while leaving clean traces unchanged.
    Prefer it whenever the data are noisy or the single-pick $P(r)$ shows the
    tell-tale short-$r$ noise spike; for clean data it self-reduces to the single
    pick, so it is always a safe default for noisy work.

---

## joint_background() { #joint_background data-toc-label="joint_background" }

```python
bg = deer.joint_background(t, V, bg_start=None, bg_end=None, dim=3.0,
                           fit_dim=False, nu_dd=deer.NU_DD, n_r=60,
                           rate_alpha=1.0, lam_pin_frac=0.5)
```

The λ-pinned joint background of [`deer_invert_joint()`](#deer_invert_joint),
stripped to return **only** the background (same dict shape as
[`background_fit()`](#background_fit)). The rate is fit on a coarse internal
distance grid (`n_r`) at a fixed regularization (`rate_alpha`): $k$ and $\lambda$
are insensitive to the $P(r)$ resolution, so this is ~30× faster than a full joint
inversion — fast enough to re-run per background-start during Mellin validation. It
backs [`deer_invert_mellin()`](#deer_invert_mellin) with `bg_engine='joint'`.

Three robustness measures versus the in-line joint fit, all aimed at the Mellin
engine (which cannot absorb a residual background the way Tikhonov hides it as
spurious long-$r$ mass):

- **λ pinned over the later, more-decayed part of the tail** (`lam_pin_frac`, the
  last 50 % of $[\text{bg\_start}, T_\max]$ by default). $\lambda$ is the
  *asymptotic* baseline ($F\to0$); pinning $\langle F\rangle = 0$ over the whole
  tail biases it high when a broad/long-$r$ component has not decayed ($\langle
  F\rangle > 0$ there), underestimating $\lambda$ and pushing $k$ too steep — a
  residual tail droop the Mellin engine cannot represent. The later window is more
  decayed, so $k$ returns near-true. (The rate-fit residual still spans the whole
  trace; `bg_end` only seeds the initial guess.)
- **Adaptive rate-fit distance cap.** $k$ is fit on both the trace-supported tight
  cap $r_\max \approx 5\,(T_\max/2)^{1/3}$ nm *and* a wider one, preferring the
  wider result **unless** widening collapses the background toward flat ($k\to0$,
  the long-$r$/background degeneracy the tight cap guards against). This stops a
  genuine broad/long-$r$ component being forced into the background (which would
  leave a Mellin-unrepresentable pedestal) while still keeping $k$ determined on
  short single-peak traces.

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
$F$ has fallen to `fit_level`·$F_0$), so a few $\delta$-wide samples cannot make it
noisy. This removes a systematic error in the recovered $F_\text{fit}$ at the echo
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
delta = deer.mellin_delta(t, F, level=0.95, floor=0.09, cap=0.12)
```

Practical Mellin split point $\delta$: the first $T>0$ where the form factor has
fallen to `level` of $F(0)$ ($F(\delta)\approx0.95$). Falls
back to the first positive sample if $F$ never drops that far.

The raw level estimate is then **clipped to `[floor, cap]`** (µs; set either to
`None` to disable, and both are clamped to the last sample). The **floor** is the
key correction for *sharp* distributions: a fast-decaying form factor crosses
`level` within a couple of samples, leaving the analytic parabolic $[0,\delta]$
echo-top anchor too narrow (the "thin parabola"), so the recovered $F_\text{fit}$
top comes out too steep and the short-$r$ density is unstable — widening $\delta$ to
$\approx 90$ ns gives the parabolic term enough low-$T$ support. The **cap**
($\approx 120$ ns) stops a slow-decaying (long-$r$) trace from over-smoothing
$P(r)$ by integrating too much of the modulation analytically. Both bounds were
tuned overlap-optimally over the synthetic benchmark (13 distributions × 4 noise
levels × 2 conditions; the floor lifts e.g. `gauss_narrow` easy from 0.90 to 0.92).

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
  Robust for DEER, whose L-curve is nearly *vertical* (the residual stays at the
  noise floor across decades of $\alpha$), so the classic corner is ill-defined
  and tends to pick a tiny $\alpha$ ⇒ spiky $P(r)$. GCV uses the (unconstrained)
  Tikhonov influence-matrix trace as the effective degrees of freedom paired with
  the NNLS residual.
- **`'curvature'`** — classic maximum-Menger-curvature L-corner.

Returns a dict with `alphas`, `rho` (residual norms), `eta` (solution norms),
`curvature`, `gcv`, `alpha_opt`, `index`, `method`, and `P` (the solution at the
chosen $\alpha$).

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
