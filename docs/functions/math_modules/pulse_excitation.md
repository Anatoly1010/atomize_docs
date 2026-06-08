# Pulse Excitation Profiles

Compute how a shaped microwave pulse acts on spins across resonance offset, by
full Bloch/propagator spin dynamics of a single $S=1/2$ — the EasySpin
`exciteprofile` approach. Because it propagates the actual dynamics (not the
linear-response / FFT approximation), it is correct for **adiabatic** pulses
(WURST, sech/tanh) as well as weak ones.

```python
import atomize.math_modules.pulse_excitation as pe
```

Pure NumPy, no hardware. Use it to design pulses (bandwidth, inversion quality),
estimate where two pulses at different carriers overlap, or build small two-pulse
sequences.

## Background

In the rotating frame of the microwave carrier a spin at resonance offset
$\Delta\omega$ sees

$$ H(t) = \Delta\omega\, S_z + \omega_1(t)\,[\cos\phi(t)\,S_x + \sin\phi(t)\,S_y], $$

with $\omega_1(t) = 2\pi\,\nu_1\,a(t)$ the instantaneous nutation (B<sub>1</sub>)
drive, $a(t)\in[0,1]$ the amplitude envelope and $\phi(t)$ the running RF phase
(carrier + chirp). The pulse is propagated **piecewise-constant**: over each short
step $dt$ the Hamiltonian is held at its mid-step value and

$$ \rho \to U\rho U^\dagger, \qquad U = \exp(-i\,2\pi\,dt\,H). $$

For a two-level system $U$ is an $SU(2)$ rotation, so instead of a matrix
exponential per (offset, step) the Bloch vector $\mathbf{M}=(M_x,M_y,M_z)$ is
rotated directly with Rodrigues' formula. This vectorises over the **whole offset
axis** at once, so a microsecond WURST profile computes in well under 0.1 s.

!!! info "Units"
    Internally everything is **GHz** (frequencies) and **ns** (time) — a product
    GHz·ns is a number of cycles. The offset axis, `nu1` and the result are in
    these base units; shape parameters in the `params` dict (`bw`, `center`, …)
    are in **MHz**. Convert MHz→GHz with `/1000` on the way in.

!!! warning "What is and isn't modelled"
    A single isolated $S=1/2$, coherent dynamics only — **no relaxation**
    (T<sub>1</sub>/T<sub>2</sub>), no B<sub>1</sub> inhomogeneity or resonator
    bandwidth, no coupled/nuclear spins. The on-resonance flip angle from
    [`flip_angle`](#flip_angle) is the pulse *area*; for swept/adiabatic pulses
    the effective rotation is offset-dependent and not a single number.

## Pulse shapes

`pe.SHAPES` lists the supported shapes. Parameter names and units follow the
common AWG (arbitrary-waveform generator) convention, so a pulse designed here
reproduces the one the hardware emits. Widths are in **ns**, frequencies in
**MHz**, `b` in **1/ns**. Every shape also reads `center` — the pulse **carrier**
frequency (MHz), which shifts the whole excitation band along the offset axis
(for WURST/sech it is the sweep centre).

| Shape | Extra `params` | Notes |
| ----- | -------------- | ----- |
| `rectangular` | `center` | constant envelope |
| `gaussian` | `sigma` (ns), `center` | $\exp[-\tfrac12((t-t_p/2)/\sigma)^2]$ — `sigma` is the Gaussian std (not FWHM) |
| `sinc` | `sigma` (ns), `center` | $\mathrm{sinc}(2(t-t_p/2)/\sigma)$, first zero at $\pm\sigma/2$ |
| `half-sine` | `center` | $\sin(\pi t/t_p)$ |
| `quartersine` | `edge` (ns), `center` | flat top, quarter-sine rise/fall over `edge` |
| `WURST` | `n`, `bw` (MHz), `center` | $1-\lvert\sin\rvert^{\,n}$ envelope, linear chirp across `bw` |
| `sech/tanh` | `b` (1/ns), `n`, `bw` (MHz), `center` | HS$n$ hyperbolic-secant adiabatic inversion; `b` truncation, `n` order |

!!! note "Sample-grid convention"
    The sech/tanh envelope and sweep are defined on a sample grid (the module
    constant `SAMPLE_RATE`, default 1.25 samples/ns = 1250 MHz), matching how an
    AWG builds the waveform; `b` is therefore tied to that sample rate. The other
    shapes are sample-rate-independent.

## `excitation_profile(shape, tp, nu1, offsets, params, dt=0.5, phi0=0.0, init=(0,0,1))` { #excitation_profile }

The excitation/inversion profile of a **single** pulse from an initial state
(default equilibrium $+z$).

| Argument | Description |
| -------- | ----------- |
| `shape` | one of `pe.SHAPES` |
| `tp` | pulse length (ns) |
| `nu1` | peak nutation / B<sub>1</sub> frequency (**GHz**); rectangular flip $=2\pi\,\nu_1 t_p$ |
| `offsets` | array of resonance offsets (**GHz**) |
| `params` | shape parameters (MHz units), see table above |
| `dt` | propagation step (ns); smaller = more accurate / slower |
| `phi0` | constant pulse phase (rad) — x/y/… |
| `init` | initial magnetization `(Mx, My, Mz)` |

Returns `Mx, My, Mz` arrays over `offsets`. `Mz` is the inversion profile;
`hypot(Mx, My)` is the transverse excitation.

```python
import numpy as np
import atomize.math_modules.pulse_excitation as pe
import atomize.general_modules.general_functions as general

offsets = np.linspace(-0.25, 0.25, 401)               # GHz  (-250..250 MHz)
Mx, My, Mz = pe.excitation_profile(
    'WURST', tp=200, nu1=0.031, offsets=offsets,       # 31 MHz B1
    params={'n': 20, 'bw': 200, 'center': 0})
general.plot_1d('WURST inversion', offsets*1e3, Mz, xname='Offset', xscale='MHz',
                yname='Mz/M0')
general.message('inversion band reaches Mz = %.2f' % Mz.min())
```

A rectangular $\pi$ pulse for comparison (carrier shifted to +60 MHz so the band
sits there):

```python
Mx, My, Mz = pe.excitation_profile('rectangular', tp=16, nu1=1/(2*16),
                                   offsets=offsets, params={'center': 60})
```

## `waveform(shape, t, tp, params)` { #waveform }

The building block: amplitude envelope and instantaneous frequency of a shape.

Returns `(a, nu)` — `a` the normalised envelope in $[0,1]$, `nu` the instantaneous
frequency (GHz) relative to the reference (constant `center` plus, for swept
shapes, the chirp). Use it to plot the pulse or feed a custom propagation.

```python
t = np.linspace(0, 200, 400)
a, nu = pe.waveform('WURST', t, 200, {'n': 20, 'bw': 200, 'center': 0})
I, Q = a*np.cos(2*np.pi*np.cumsum(nu)*(t[1]-t[0])), a*np.sin(2*np.pi*np.cumsum(nu)*(t[1]-t[0]))
```

## `flip_angle(shape, tp, nu1, params, dt=0.5)` { #flip_angle }

On-resonance flip angle (rad) — the integral of the envelope, $2\pi\,\nu_1\!\int a\,dt$.
Exact for non-swept shapes; the nominal area for chirp/adiabatic pulses.

```python
fa = pe.flip_angle('gaussian', tp=40, nu1=0.02, params={'sigma': 10})
general.message('flip = %.1f deg' % np.degrees(fa))
```

## Building sequences

Two lower-level helpers let you chain pulses for multi-pulse experiments.
`excitation_profile` is just `propagate_pulse` on a tiled equilibrium state.

### `propagate_pulse(M, shape, tp, nu1, offsets, params, dt=0.5, phi0=0.0)` { #propagate_pulse }

Apply one pulse to an existing Bloch-vector array `M` of shape
`(len(offsets), 3)`; returns the new array (input not mutated).

### `free_evolution(M, offsets, tau)` { #free_evolution }

Free precession for `tau` ns: rotate each offset about $+z$ by
$2\pi\,\Delta\omega\,\tau$ (same sign convention as a zero-amplitude pulse).

```python
# Hahn echo: pi/2_x  --  tau  --  pi_y  --  tau, transverse magnetization vs offset
offsets = np.linspace(-0.05, 0.05, 401)
M = np.tile([0., 0., 1.], (offsets.size, 1))                       # equilibrium +z
M = pe.propagate_pulse(M, 'rectangular', 16, 1/(4*16), offsets, {'center': 0})   # pi/2 x
M = pe.free_evolution(M, offsets, 200.0)
M = pe.propagate_pulse(M, 'rectangular', 16, 1/(2*16), offsets, {'center': 0},
                       phi0=np.pi/2)                                # pi y
M = pe.free_evolution(M, offsets, 200.0)
general.plot_1d('echo profile', offsets*1e3, np.hypot(M[:, 0], M[:, 1]),
                xname='Offset', xscale='MHz', yname='|Mxy|')
```

!!! note "Carrier phase across a delay"
    Each pulse's phase is referenced to its **own** start. For a continuous-LO
    instrument where pulse $k$ at carrier $\nu_{0k}$ keeps accumulating phase
    during the delay, add the absolute-time term yourself:
    `phi0 += 2*pi*nu0_k*(t_start_k)` (in GHz·ns). This only matters when a later
    pulse sits at a **non-zero carrier** (e.g. a DEER pump).
