# EPR automation (epr_auto)

`epr_auto` is a YAML protocol runner for unattended pulsed-EPR tune-up and
measurement on the endstation's X-band pulsed machine. Instead of clicking
through the phasing, field, and acquisition tools by hand for every sample, you
declare the run as a list of steps in a text file; the runner tunes the
spectrometer, judges every result, adapts the scan count to a signal-to-noise
target, and records a manifest of what it did.

A protocol is a short, diffable description of intent. Each step names a
primitive — set the temperature, find the line, calibrate the pulses, zero the
phase, measure a relaxation curve — with per-step parameter overrides. The
runner executes them in order, applies a pass/fail judge to each result, carries
calibrations forward (a measured field or phase feeds later steps), and pauses
or notifies according to the protocol's autonomy policy.

It ships only with [Atomize_ITC](https://github.com/Anatoly1010/Atomize_ITC),
the EPR-endstation variant of Atomize — not with plain Atomize and not from
PyPI. The machine it drives is described on the [endstation page](../endstation.md).

## How it relates to the GUI tools

`epr_auto` does not re-implement the spectrometer. Its engine reuses the AWG
phasing tool's own acquisition worker (the `Worker` class from
`awg_phasing_insys.py`), the same hardware-validated scan loop the GUI pickles
when you press Start. The engine only rebuilds the argument tuples the GUI would
have built from a preset, so an engine run is argument-identical to a
GUI-launched run of the same `*.phase_awg` preset. Because the acquisition path
is unchanged, a run started from a terminal while the Atomize GUI is open plots
live into that GUI exactly as an interactive scan does — the runner reaches the
same `LivePlot` socket. This is also why the command-line runner changes its
working directory into `libs/` before touching hardware: the Insys FPGA driver
reads its `brd.ini` / `exam_adc.ini` relative to the current directory at
instantiation time.

## A protocol at a glance

```yaml
sample: RO2411
autonomy: checkpointed        # supervised | checkpointed | autonomous
notify: none                  # telegram: notify on checkpoint/finish/abort

steps:
  - tune.power_for_length:    # coarse: rotary vane sets the power regime
      target_length: 32 ns
      amplitude: 95
  - field.edfs:               # find the line, center from h*nu/(g*mu_B)
      range: auto
      pick: max
  - tune.echo_window          # integration window from the averaged echo
  - tune.auto_phase           # zero the signal phase (runs after the window)
  - tune.pi_calibration:      # fine: per-pulse AWG amplitude sweep
      mode: amplitude
      retries: 1
```

Steps with no parameters are written as a bare name; everything else is a
key/value block. Full step and parameter details are on the
[Step reference](steps.md) page and are also available offline from
`epr-auto steps`.

## Current status

The runner is implemented and dry-run-verified end to end; on-hardware
commissioning is in progress.

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — every step is implemented and
    verified in dry-run (`--test`) only. Always dry-run a protocol first, and
    keep the first live sessions in `supervised` autonomy with an operator
    present.

## Where to go next

| Page | What it covers |
| ---- | -------------- |
| [Quickstart](quickstart.md) | Install, validate, and dry-run a protocol |
| [Writing protocols](protocols.md) | The YAML schema, autonomy, `foreach` series, value substitution |
| [Step reference](steps.md) | Every step, its parameters, defaults, and constraints |
| [The tune-up chain](tuning.md) | How the calibration steps compose and invalidate each other |
