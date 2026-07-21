# Presets

Every acquisition `epr_auto` runs is built from a `*.phase_awg` **preset** —
the same saved pulse-sequence file the AWG phasing tool uses interactively. A
protocol step names a preset; the runner loads it, applies the calibrations
earlier steps measured, and rebuilds the exact worker arguments the GUI would
have built from it. This page is what a protocol author needs to know about
those presets: where they come from, which sweep-type family each step
expects, how the automation's demodulated 1-D acquisition mode relates to the
preset's "Shift Offset" checkbox, and what the runner overrides at build time
versus what it never touches. For the individual
step parameters see the [Step reference](steps.md); for how the tuning
results flow between steps see [The tune-up chain](tuning.md).

## Where presets come from

A preset is not written by hand. It is saved from the endstation's
interactive tools and lives as a `*.phase_awg` file:

- the **AWG phasing tool** (`awg_phasing_insys.py`) — the Save button of the
  same window you tune a sequence in;
- the **Sequence Calculator**, which builds a sequence from inter-pulse
  delays and pushes it out as a `*.phase_awg` preset.

The shipped presets live in `atomize/control_center/experiments/`
(`hahn_echo_4s.phase_awg`, `ed_4s.phase_awg`, `ampl_4s.phase_awg`,
`rabi_echo_4s.phase_awg`, `inversion_recovery_echo_4s_log.phase_awg`, …) and
are the defaults the steps fall back to. A step's `preset:` parameter takes
either a bare filename or a path: a **bare name resolves against the
protocol's own directory first, then the shipped experiments directory**, so
a protocol can carry a sample-specific preset next to itself and have it
shadow the shipped one. A name that resolves nowhere is a load-time error
that lists both places it looked, so `epr-auto validate` catches a missing
preset at your desk.

!!! note
    The engine reads the preset with the **same parser the phasing GUI uses**
    — identical line indices, grid snapping, and phase-cycle expansion — so
    a preset that runs in the GUI runs identically under the runner. This
    page stays at "what a protocol author must know"; the file format itself
    is the phasing tool's, not the runner's, and is documented with that
    tool.

## Sweep-type families

Every preset carries a **sweep type**, and each step accepts only the
family that matches the physics it measures. Handing a step the wrong
family is rejected before any hardware moves — in the `--test` pre-flight as
well as live — with a message naming the family it wanted:

| Step | Sweep type it needs | Shipped default |
| --- | --- | --- |
| `tune.auto_phase` | Linear Time (echo) | `hahn_echo_4s.phase_awg` |
| `tune.echo_window` | Linear Time (echo) | `hahn_echo_4s.phase_awg` |
| `tune.rep_rate` | Linear Time (plain echo) | `hahn_echo_4s.phase_awg` |
| `tune.pi_calibration` `mode: amplitude` | Amplitude | `ampl_4s.phase_awg` |
| `tune.pi_calibration` `mode: length` | Linear Time (length nutation) | `rabi_echo_4s.phase_awg` |
| `tune.power_for_length` | Linear Time (length nutation) | `rabi_echo_4s.phase_awg` |
| `field.edfs` | Field | `ed_4s.phase_awg` |
| `exp.t2` | Linear Time (moving-echo Hahn) | `hahn_echo_4s.phase_awg` |
| `exp.t1` | Log Time (inversion recovery) | `inversion_recovery_echo_4s_log.phase_awg` |

Two of these carry an extra structural requirement beyond the sweep type:

- **`exp.t2` needs a *moving-echo* Linear Time preset** — the DETECTION pulse
  must itself move with the tau sweep, so that the saved axis is the physical
  evolution time `2·tau`. A Linear Time preset whose detection pulse is frozen
  (a pump-only sweep such as a 4-pulse DEER) is rejected: `the DETECTION pulse
  does not move (st_inc == 0) — not a moving-echo (Hahn) T2 preset`.
- **`tune.pi_calibration` needs exactly one swept pulse** — the pulse the
  sweep moves (nonzero start increment in an Amplitude preset, nonzero length
  increment in a length nutation). Zero or more than one is rejected.

`tune.rep_rate` additionally *warns* (it does not reject) when handed a Log
Time or Amplitude preset: those sweeps flip the echo sign across their own
points, which cancels the amplitude metric rep-rate fits, so the warning
names the mistake before the coherence gate rejects the run. Use a plain
Linear Time echo sweep for it.

## The demodulated 1-D acquisition mode

Every automation acquisition runs in **demodulated-integral (1-D) mode**: the
worker routes each readout through the digitizer's demodulation — against the
DETECTION pulse's frequency, with the preset's zero/first/second-order phase,
integrated over the integration window — and hands the engine a single complex
I/Q point per sweep step, i.e. the 1-D curve the runner fits and judges.

In the AWG phasing tool this mode is the **"Shift Offset"** checkbox, saved in
the preset file as the line `IQ Correction:  2` (2 is the checkbox's checked
state). For an *interactive* GUI run the checkbox is a display choice: checked
gives the integrated 1-D plot, unchecked gives the full 2-D time-resolved
display. For an *automation* run it is not a requirement you have to remember —
**the runner enables the mode itself at build time**, whichever way the preset
was saved. If the preset was saved with Shift Offset off, it logs that it is
turning the mode on for the run:

```text
preset saved with Shift Offset off — enabling the demodulated 1-D mode (iq_cor 1) for the automation run
```

so nothing about a preset's Shift Offset state blocks it from being used in a
protocol. Every shipped preset already carries the mode.

## The fine timing grid

Presets snap their pulse timings onto a grid. The default is a 3.2 ns tick;
a preset saved on the AWG's finer one-DAC-sample grid carries a trailing
`AWG grid:  0.8` line, and the runner honours it exactly as the GUI does —
the P2…P9 timings and the detection start snap to 0.8 ns while the detection
length (the ADC window) stays on the 3.2 ns tick. There is nothing to
configure: whichever grid the preset was saved on is the grid the runner
uses. It matters only in that a calibration measured on one preset is
grid-quantized to the target preset's grid when it is transferred (see
[apply_cal](tuning.md#calibration-flow-and-invalidation)).

## What the runner overrides, and what it leaves alone

A preset is a starting point, not the final word. When a step builds its
worker arguments it **overrides** a handful of the preset's stored values
from the protocol and from the calibrations earlier steps measured — while
leaving everything about the pulse geometry, phases, and correction settings
exactly as the preset saved them.

The runner overrides:

- **the sweep extent and point count** from the step's own parameters —
  `points`, `scans`, and the sweep axis (`tau_start`/`tau_step` for T2,
  `t_start`/`t_end` for T1, `range`/`span` for `field.edfs`,
  `step`/`points` for the pi-calibration sweep);
- **the integration window** — an earlier `tune.echo_window` result replaces
  the preset's stored Window left/right on every later acquisition (unless a
  step sets `window: preset` to pin the preset's own values). The window is
  carried relative to the DETECTION pulse start, so it transfers across
  presets whose echo sits at a different absolute delay;
- **the demodulator zero-order phase** — an earlier `tune.auto_phase` result
  replaces the preset's stored Zero order;
- **the working field** — an earlier `field.edfs` / `field.set` result
  replaces the preset's stored Field;
- **the repetition rate** — a step's `rep_rate:` (a number, or `auto` pulling
  in the `tune.rep_rate` recommendation) replaces the preset's Rep rate;
- **the pulse amplitudes (or lengths)** — `apply_cal` patches the calibrated
  π / π₂ value into the mapped pulse slots (and re-scales the soft detection
  pair), transferring the fine calibration into the experiment preset. This
  is the one override that reaches *inside* the pulse table; it is described
  in full under
  [Calibration flow](tuning.md#calibration-flow-and-invalidation).

The runner does **not** touch anything else the preset carries: the pulse
types, their relative delays and geometry ratios, the phase-cycle notation,
the detection frequency, the resonator-correction model, the AWG intermediate
frequency, or the number of averages. Those are the sequence's identity —
change them by editing and re-saving the preset in the phasing tool, not from
a protocol.

!!! note
    The overrides above come from two sources, and it is worth keeping them
    apart. The **step parameters** (points, scans, tau, field range, …) are
    explicit in your YAML. The **session overrides** (window, phase, field,
    rep-rate, calibration) are *measured* by earlier steps and flow forward
    automatically — which is exactly why a calibration is invalidated when a
    later action makes it stale. That machinery is the subject of
    [The tune-up chain](tuning.md).

## Next steps

- [The tune-up chain](tuning.md) — how the measured overrides (window,
  phase, field, rep-rate, `apply_cal`) are produced, flow forward, and are
  invalidated.
- [Examples](examples.md) — two annotated protocols that use the shipped
  presets end to end.
- [Step reference](steps.md) — every step's `preset:` default and the rest
  of its parameters.
