# The tune-up chain

Before a relaxation measurement can run unattended, the spectrometer has to be
brought to a working point: enough microwave power that the pulses reach a
π rotation, the magnet on the resonance line, an integration window over the
echo, the receiver phase zeroed, and the pulse amplitudes calibrated. `epr_auto`
does this as an ordered chain of tuning steps, each one measuring a quantity,
judging it, and storing the result for the steps that follow. This page walks
through that chain in the order the shipped `protocols/tune_up.yaml` runs it and
explains the physics behind each rule. For the parameter tables see the
[step reference](steps.md); for how retries and `on_fail` wrap each step see
[Writing protocols](protocols.md).

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — everything below is implemented
    and verified in dry-run (`--test`) only. Follow the whole chain end to
    end with `epr-auto run protocols/tune_up.yaml --test` before trusting it
    with the magnet, and keep the first live sessions in `supervised`
    autonomy with an operator present.

The canonical order is:

```text
[temperature] -> coarse power -> field -> echo window -> phase -> fine calibration
```

Each stage depends on the ones before it. Power is set first because every
later measurement needs pulses that rotate the spins; the field search needs
those pulses to produce an echo; the echo window and phase are measured on that
echo; and the fine amplitude calibration is meaningful only once the window and
phase are fixed. A temperature prologue (`temp.set` / `temp.wait`) is optional
and, when present, runs before everything else so the whole tune-up happens at
the measurement temperature.

## Coarse power stage — `tune.power_for_length`

```yaml
- tune.power_for_length:
    target_length: 32 ns
    amplitude: 95
    checkpoint: true
```

The microwave power is set mechanically, by a rotary vane in the bridge that
attenuates the drive by a settable number of decibels. The step's job is to
find the vane position at which a π pulse lands at the requested length while
the AWG amplitude is held at a fixed, comfortable value (95 % of full scale in
the example), leaving headroom for the fine calibration to work in.

It measures the current π length with a length nutation at the held amplitude,
then moves the vane and repeats. The move size comes straight from the physics:
the B₁ field the spins see scales as `10^(-dB/20)` with vane attenuation, and a
π rotation needs a fixed B₁·τ product, so to bring a measured π length to the
target the attenuation is stepped by

```text
dB += 20 * log10(target_length / measured_length)
```

which would hit the target in one move if B₁ were exactly this ideal function of
attenuation; a few iterations (default four) absorb the residual nonlinearity of
the real vane.

Two mechanical details matter. The vane is always driven to its target *from
above* — if the new setting is a smaller attenuation, the step first overshoots
to a higher attenuation and comes back down — so that every approach compresses
the same gear backlash the same way and the position is repeatable. And after
commanding a move the step waits out the mechanical travel (the device's own
calibration curve gives roughly 36 ms per motor step) before the next
acquisition, because that acquisition runs in a separate worker process and the
vane must be at rest first.

Because a vane move changes B₁ for *everything*, it invalidates the receiver
phase and the fine amplitude calibration — see
[Calibration flow and invalidation](#calibration-flow-and-invalidation) below.
This is why the coarse stage runs once, near the top of a protocol, and why the
example marks it `checkpoint: true`: a vane move is the kind of action worth
confirming before it proceeds.

Each iteration is hard-gated on its own nutation's `echo_snr`: if the length
nutation is in the noise the step aborts rather than commanding a vane move from
an argmin of noise, and the final iteration's calibration judges ride along into
the step's judge list and the manifest. If the step runs out of iterations
without hitting the target it does **not** move the vane past the last
measurement — the reported attenuation and π length always describe the state the
vane is actually left in.

Because a cold start has no echo to nutate on until the magnet is on the line,
`epr-auto validate` warns when a `tune.power_for_length` (like `tune.auto_phase`
and `tune.pi_calibration`) appears before any `field.*` step; the warning is
informational — tuning at a manually pre-set field is legitimate.

## Signal search — `field.edfs` with `range: auto`

```yaml
- field.edfs:
    range: auto
    pick: max
    checkpoint: true
```

With power set, the run finds the resonance line by an echo-detected field
sweep. `range: auto` turns the step into a blind signal search: it predicts the
centre field from the resonance condition at the **true observation frequency**
— the synthesizer/LO readout minus the preset's DETECTION-pulse intermediate
frequency (this bridge upconverts lower-sideband, so the spins see
`ν = ν_LO − ν_IF`):

```text
B[G] = 0.714477 * ν[MHz] / g   (+ offset)
```

then sweeps ± a span (250 G by default) around it; the log shows the arithmetic
(`LO 9750 MHz - IF 50 MHz = ...`). The magnet on this endstation is not
absolutely calibrated, so a known setup shift is passed as `offset` and folded
into the predicted centre — `offset` is the *magnet-calibration* shift only,
the IF contribution is already accounted for. The step reports back the
measured line-minus-centre distance as `shift_g`, which is exactly the value to
feed in as `offset` next time. `pick: max` takes the working field at the
magnitude maximum of the sweep and sets the magnet there.

The search escalates once. If the echo-SNR judge fails — no line above the noise
floor in the first span — the span doubles and the sweep re-runs a single time.
If it fails again the step stops without moving the magnet to a noise maximum and
surfaces a diagnosis naming what to check:

- **"weak line found"** — something did clear the weak-line threshold near a
  named field, so the advice is to add scans or narrow the range around it;
- **"flat everywhere"** — nothing anywhere in the range, so the advice points at
  the resonator tuning, temperature, g-factor guess and sample position.

An explicit `range: [start, end]` skips the prediction and sweeps the given
window; `pick: value` names a field independent of the echo, in which case a
weak sweep neither escalates nor blocks — the magnet is still parked at the
requested field.

## Echo window — `tune.echo_window`

```yaml
- tune.echo_window
```

The integration window is the time gate over which the echo is summed into a
single complex point. `tune.echo_window` sets it from an averaged echo trace: it
finds the centre at the maximum of the smoothed magnitude `|V(t)|`, measures the
echo's full width at half maximum, and opens a window of `FWHM × factor` (factor
2 by default) around the centre, rounding the edges outward onto the ADC grid and
clamping them to the acquisition window. The magnitude is smoothed with a fixed
physical box (~3 ns), not a fraction of the trace length, so measuring the FWHM
does not inflate the width of a short, sharp echo.

The window is stored **relative to the DETECTION pulse start**, not as an
absolute time. That is the same frame the preset's own "Window left/right" live
in, so the measured window transfers to any later preset whose echo sits at a
different absolute delay — a T₂ decay and a T₁ recovery built from the same
detection block reuse one window without re-measuring it.

The judge checks that the echo is fully resolved inside the trace: an FWHM edge
sitting on the trace boundary means the acquisition window is cutting the echo,
and the result is flagged.

This step must run **before** `tune.auto_phase`, because auto-phase integrates
the signal over exactly this window to measure the residual phase. A phase
measured over a wrong or unset window is meaningless, which is why the chain
fixes the window first.

## Receiver phase — `tune.auto_phase`

```yaml
- tune.auto_phase
```

The digitizer demodulates the echo against a reference whose zero-order phase
offset must be tuned so the signal lands on the real axis. `tune.auto_phase`
acquires a short echo run, integrates it over the echo window, and measures the
residual signal phase by the principal-axis method (`auto_phase_zero` in the FFT
module — a sign-blind Σ S² estimate that phases a bipolar trace correctly). The
demodulator rotates by `exp(-i·zero_order)`, so the correction is a direction:

```text
zero_order_new = zero_order_used - phi_residual
```

The step acquires 16 sweep points by default. The `phase_coherence` judge that
gates it has an N-aware pass floor, `max(0.7, 3/√n)`, so a trace of fewer than
about ten points can never clear the noise floor — 16 gives the gate something
to work with while staying a quick acquisition.

The updated zero-order is stored and flows into every later acquisition's build.
Auto-phase also records the temperature it was measured at, which is what lets a
later temperature move decide whether the phase is still valid.

## Fine calibration — `tune.pi_calibration`

```yaml
- tune.pi_calibration:
    mode: amplitude
    retries: 1
```

The coarse stage put a π near the target *length* at a fixed amplitude; the fine
stage pins the pulse amplitudes precisely. The nutation presets are
inversion-detection sequences — the pulse being swept precedes a fixed selective
echo pair, so the echo amplitude goes as `cos(θ)` of the swept pulse, with π at
the first minimum and π/2 at the zero crossing.

The rotation angle is fitted as a quadratic in the swept quantity,

```text
theta(x) = b*x + s*x^2
```

The quadratic term absorbs amplifier compression — the rotation slows slightly
as the amplitude climbs — so `amp(π)` and `amp(π/2)` come out **independently**
from the fit rather than forcing the ideal `amp(π) = 2·amp(π/2)`. (Because
compression only ever slows the rotation, the fitted chirp is bounded to a
physical range; without that bound the fit has a spurious accelerating-chirp
basin that wrecks the π/2 root.) The π extremum is de-biased against the fit's
envelope/decay degeneracy with a model-free parabola vertex at the observed
minimum, and π/2 is then re-fitted with the angle map anchored at that measured
π, which makes the π/2 root as reliable as π itself.

`mode: amplitude` sweeps the AWG amplitude in percent at fixed length — the usual
fine calibration; `mode: length` runs a linear-time length nutation and is what
the coarse stage uses internally. After an amplitude calibration the soft
detection pair (the selective refocusing/detection pulses, recognised as the
active pulses at ≥ 2.5× the swept pulse's length) is re-scaled onto the
calibrated value by the flip-angle relation — the on-resonance rotation goes as
`amp · L · ⟨envelope⟩`, so the amplitude scales by the calibrated `L·⟨envelope⟩`
over the target slot's, the AWG amplitude being nearly linear. The envelope area
factor `⟨envelope⟩` is 1 for a SINE pulse and the mean of the peak-referenced
envelope for a shaped one (0.543 for the shipped GAUSS); a transfer *across*
envelope shapes for which no area factor exists — the adiabatic WURST/SECH — is
refused rather than mis-scaled. `refine: true` re-runs the nutation once with the
re-scaled pair patched in, as insurance on a new sample.

If the fitted π amplitude runs into the rails — π unreachable within the sweep,
or above the amplitude ceiling — the step fails on the `amplitude_rails` judge;
an out-of-sweep π/2 rails the step the same way, so a π/2 outside the swept range
is never shipped. A `mode: length` calibration is rail-checked identically by a
`length_rails` judge. The `retries: 1` in the example grants one plain
re-attempt; on a **rail**
failure specifically, the runner additionally re-runs the coarse power stage
declared earlier in the protocol (and the auto-phase steps tuned after it, which
the vane move invalidates) once and retries the calibration, so a sample that
needs more power can recover without operator intervention. This fallback exists
only when the protocol itself declared a `tune.power_for_length` step earlier;
with no coarse stage to fall back to, the rail failure is reported as-is.

## Repetition rate — `tune.rep_rate`

`tune.rep_rate` is not in `tune_up.yaml` (it belongs before a relaxation run
rather than in the basic tune-up), but it is part of the same chain and feeds the
experiment steps. It runs one quick echo acquisition per rate on a log grid,
slowest first, and fits the steady-state saturation of the sequence repeated
every period `T = 1/rate`:

```text
A(T) = A0 * (1 - exp(-T / T1_eff))
```

The fitted `T1_eff` is the recovery time the sequence actually sees. From it the
step recommends a working rate in one of two modes:

- **`quantitative`** (default): period = `factor × T1_eff`, with `factor` 5
  giving under 1 % residual saturation — the right choice when the saturation
  itself would bias a relaxation measurement;
- **`sensitivity`**: period = `1.26 × T1_eff`, the `A(T)/√T` optimum that
  maximises echo per unit √time. It accepts partial saturation and is fine for
  tuning and EDFS but **not** for quantitative relaxation runs.

A recommendation is never extrapolated above the fastest tested rate. Two edge
cases are handled explicitly: if the amplitude is flat across the whole grid the
sequence never saturates even at the fastest rate, so that rate is reported as
safe with `T1_eff` below the grid; if even the slowest rate is still saturated
the fit is an extrapolation beyond the grid and no recommendation is stored —
the advice is to extend `rate_min` lower. The stored recommendation is what
`exp.t1` / `exp.t2` pick up with `rep_rate: auto`.

The step refuses to recommend a rate off something that is not an echo: all
acquisitions concatenated must share one phase (the same resultant-length
gate auto-phase uses), and on a strongly saturating curve the amplitude
deviations must additionally lie on that phase axis — a constant
instrumental phasor (LO leakage surviving the phase cycle) can fake the
shared phase but not the structure, and is rejected with a note naming it.
The preset should be a plain echo sweep: a Log Time (inversion-recovery) or
Amplitude (nutation) preset flips the echo sign across its own sweep points,
which cancels the step's amplitude metric — the step warns up front when it
sees one, naming the mistake before the coherence gate rejects the run.

The `--test` dry-run pre-flights **every** rate of the grid, so a `rate_max`
whose period is shorter than the sequence fits in fails the dry-run rather than
aborting a live run.

## Calibration flow and invalidation

Every tuning step writes its result into the session state, and every later
build reads it back ([Presets](presets.md) lists which preset fields each
override replaces). This is how a value measured once propagates without being
repeated:

| Stored by | Flows into |
| --------- | ---------- |
| `tune.echo_window` | the integration window of every later acquisition |
| `tune.auto_phase` | the demodulator zero-order of every later acquisition |
| `field.edfs` / `field.set` | the working field of every later acquisition |
| `tune.rep_rate` | `exp.*` steps that set `rep_rate: auto` |
| `tune.pi_calibration` | `exp.*` steps via `apply_cal` |

The pulse calibration reaches the experiment presets through the `apply_cal`
parameter on the `exp.*` steps. By default the runner infers which pulses to
patch from the preset itself: among the hard microwave pulses it reads the two
amplitude levels the preset already carries and maps the higher to π and the
lower to π/2 (or, for an equal-amplitude preset, the 2×-longer pulse to π). When
that is ambiguous you give an explicit map, e.g. `apply_cal: {P2: pi2, P3: pi}`;
`apply_cal: none` opts a step out of patching entirely and runs the
preset-stored amplitudes. Patched amplitudes are transferred to pulses of a
different length — or a different envelope shape — by the same flip-angle
relation the detection pair uses (the amplitude scales as the calibrated
`L·⟨envelope⟩` over the target slot's), so a calibration measured at one pulse
length applies correctly at another; a transfer across envelope shapes with no
area factor (adiabatic WURST/SECH) is refused. A calibration that itself railed
is rejected here rather than shipped.

Calibrations do not survive changes that physically invalidate them. The rules
are grounded in measured behaviour of this endstation:

| Change | Dropped | Kept | Why |
| ------ | ------- | ---- | --- |
| Rotary-vane move | `auto_phase` **and** `pi_calibration` | `rep_rate` | The vane changes B₁ for everything: both the phase and the amplitude calibration are stale. T₁ does not depend on B₁, so the rep-rate recommendation holds. |
| Temperature setpoint move (beyond `rephase_delta`) | `auto_phase` **and** `rep_rate` | `pi_calibration` | Temperature detunes the resonator, so the demod zero-order drifts — a measured temperature series showed a monotonic swing of roughly a degree of phase per kelvin, which sets the small default `rephase_delta`. And T₁ — the whole basis of the `tune.rep_rate` recommendation — changes strongly with temperature, often by orders of magnitude across a cryostat range. B₁ is untouched, so the amplitude calibration still holds. |
| Field move | nothing | all three | A field move changes neither the resonator tuning nor B₁, and its effect on T₁ is minor (it appears only in systems with two different spins), so no calibration is affected. |

The temperature rule compares the new setpoint against the temperature each
value was actually measured at (auto-phase and rep-rate both record it, and
they are checked independently — in a series they may have been tuned at
different points); only a move larger than `rephase_delta` drops a value, so
small excursions do not force a needless re-tune. Dropping is not
re-measuring: once the stored phase is gone, later acquisitions fall back to
the preset's own zero-order — and once the stored rate is gone, a later
`rep_rate: auto` step fails loudly instead of running saturated. A protocol
that changes temperature should therefore follow its `temp.wait` with a fresh
`tune.auto_phase` (and a fresh `tune.rep_rate` if the experiments use
`rep_rate: auto`) — the invalidation guarantees a stale value is never
silently carried, and the fresh step measures the new one.

## Running the chain

```bash
epr-auto run protocols/tune_up.yaml --test
```

This exercises the whole chain end to end with no hardware attached: every step
pre-flights its exact worker arguments under a test-mode device, the judges and
the state flow run, and canned measurements stand in for the acquisitions. It is
the pre-flight to run after editing a protocol. The endstation itself is
described on the [endstation page](../endstation.md); the steps used here, with
their full parameter tables, are in the [step reference](steps.md).
