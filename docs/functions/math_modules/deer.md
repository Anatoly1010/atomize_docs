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
(ill-posed); this module solves it with **Tikhonov regularization +
non-negativity** (NNLS), the regularization weight $\alpha$ chosen at the
**L-curve** corner.

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
                       reg_order=2, nu_dd=deer.NU_DD, scan_lcurve=True)
```

The full one-call pipeline: background-correct $V(t)$, build the kernel, and
invert to $P(r)$ by Tikhonov + NNLS. This is what most users want.

- **`t`, `V`** — time axis (µs) and the real DEER trace $V(t)$.
- **`r`** — distance grid (nm). `None` uses [`default_r_axis()`](#default_r_axis)
  (1.5–8 nm, 200 points).
- **`bg_start`, `bg_end`** — background-fit window (µs). `bg_start=None` defaults
  to the midpoint of the trace; `bg_end=None` fits to the end. See
  [`background_fit()`](#background_fit).
- **`dim`, `fit_dim`** — fractal background dimension (3 = homogeneous 3D); set
  `fit_dim=True` to float it.
- **`alpha`** — regularization weight. `None` selects it at the L-curve corner.
- **`alphas`** — the L-curve scan grid (default `np.logspace(-4, 1, 25)`).
- **`reg_order`** — derivative order of the smoothing operator $L$ (default 2).
- **`scan_lcurve`** — when `True` (default) the L-curve is always computed for
  display, even if an explicit `alpha` is given.

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
| `kernel` | The dipolar kernel matrix $K$ |
| `alpha` | The regularization weight used |
| `l_curve` | The [`l_curve()`](#l_curve) result dict (or `None`) |
| `background` | The [`background_fit()`](#background_fit) result dict |
| `lambda`, `k`, `dim` | Modulation depth, background decay rate, dimension |

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
normalized so $V(0) = 1$; the tail window is fit to
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
lc = deer.l_curve(K, F, alphas, L=None)
```

L-curve scan: solution norm vs residual norm over `alphas`, with the corner
picked by maximum Menger curvature in log–log space. Returns a dict with
`alphas`, `rho` (residual norms), `eta` (solution norms), `curvature`,
`alpha_opt`, `index`, and `P` (the solution at the corner).

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
