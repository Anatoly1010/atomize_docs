# Troubleshooting

What the common failures look like and what they mean. The messages here are
quoted verbatim from the runner — searching this page for the exact line you
saw is the fastest way in. Failures fall into four groups: **load-time**
errors caught before anything runs, **judge** failures that reject a step's
data quality, **environment** problems around locks and the GUI, and
**warnings** that are not failures at all. For the concepts behind them see
[Writing protocols](protocols.md), [The tune-up chain](tuning.md), and
[Presets](presets.md).

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — the failure paths below are
    implemented and exercised in dry-run (`--test`). Always dry-run a
    protocol first.

## The `--test`-first workflow

Most mistakes never need hardware to find. Two commands catch them at your
desk:

```bash
epr-auto validate my_protocol.yaml      # parse + range-check, no run
epr-auto run my_protocol.yaml --test    # full dry-run against test-mode devices
```

`validate` resolves every preset, range-checks every parameter, and expands
`foreach` blocks. `run --test` goes further: it walks the whole protocol
against test-mode devices, pre-flighting each step's exact worker arguments —
so an illegal pulse geometry or an out-of-range setting is rejected in the
dry-run, not at the bench. **Always dry-run before a live session.**

## (a) Load-time errors

A protocol that fails to load is reported as `INVALID: <file>:<line>:
<message>` on stderr, and nothing runs. The line number points at the
offending step. These come from the schema and parameter validators; the
message names the fix.

**Structure and top-level keys**

```text
INVALID: my.yaml:1: unknown top-level key 'sampl' (allowed: sample, autonomy, output, notify, steps)
INVALID: my.yaml:1: 'sample' (non-empty string) is required
INVALID: my.yaml:1: 'steps' (non-empty list) is required
```

**A mis-indented step** — the single most common structural mistake is
under-indenting a step's parameters so YAML reads them as sibling list
entries:

```text
INVALID: my.yaml:7: each step must be a single "name: {params}" mapping (check the indentation of the parameters)
```

**Unknown step or parameter** — both name the valid alternatives:

```text
INVALID: my.yaml:5: unknown step 'tune.autophase' (available: exp.t1, exp.t2, field.edfs, field.set, temp.set, temp.wait, tune.auto_phase, tune.echo_window, tune.pi_calibration, tune.power_for_length, tune.rep_rate)
INVALID: my.yaml:9: [exp.t2] unknown parameter 'taus' (valid: apply_cal, max_duration, points, preset, rep_rate, scans, target_snr, tau_start, tau_step, window)
```

**A bad parameter value** — the message is prefixed with the step and
parameter:

```text
INVALID: my.yaml:9: [exp.t2] points: required parameter is missing
INVALID: my.yaml:9: [exp.t2] tau_start: expected a "<value> <unit>" string, got 300
INVALID: my.yaml:11: [field.edfs] value: expected a "<value> <unit>" string, got 3318
```

Note the last two: a **bare number where a unit string is expected** is
rejected — times and fields are always `"<value> <unit>"` strings
(`300 ns`, `3318 G`).

**A preset that resolves nowhere** — the message lists every directory it
searched. For an explicitly named preset that is the protocol's own directory,
the user preset directory (`atomize/epr_auto/presets/`), and the shipped
experiments directory (a step *default* is searched in the shipped directory
only):

```text
INVALID: my.yaml:5: [exp.t2] preset: preset file 'my_echo.phase_awg' not found (looked in: /home/you/protocols/my_echo.phase_awg , /home/.../epr_auto/presets/my_echo.phase_awg , /home/.../control_center/experiments/my_echo.phase_awg)
```

**Cross-parameter checks** fire after the individual parameters validate:

```text
INVALID: my.yaml:9: [exp.t1] t_start must be < t_end, got 500 ns .. 300 ns
INVALID: my.yaml:5: [field.edfs] pick: value requires the 'value' parameter
INVALID: my.yaml:5: [tune.rep_rate] rate_min must be < rate_max, got 2000 .. 20 Hz
```

**`foreach` and substitution** errors:

```text
INVALID: my.yaml:12: foreach: unknown key 'vars' (allowed: var, values, steps, on_fail)
INVALID: my.yaml:14: [field.set] value: unresolved $Bfield (foreach defines: $B)
INVALID: my.yaml:12: foreach cannot be nested
```

An **`output:` template** typo is caught at load, not at first save:

```text
INVALID: my.yaml:4: 'output' template: only {date} and {sample} placeholders are supported ('run')
```

## (b) Judge failures

In a live run, every tuning, field and experiment step judges its own result.
A failed **hard** judge aborts the step and feeds the retry / `on_fail`
machinery; **advisory** judges only warn. The failure is logged as

```text
      step failed (attempt 1): [FAIL] <judge>: score=<x> (<details>)
```

so the leading `[FAIL] <judge>:` tells you which gate rejected the data. (In a
dry-run every judge is logged but none abort — `--test` shows the diagnostics
without stopping.)

**`echo_snr` — the signal is in the noise.** The default pass floor is SNR 3
(pure noise scores ~1):

```text
[FAIL] echo_snr: score=2.1 (min_snr=3, noise_sigma=0.04)
```

On the relaxation steps (`exp.t1`, `exp.t2`) `echo_snr` is *advisory* — a
noisy-but-valid decay must not be thrown out on SNR alone, so `relaxation_fit`
is their hard gate — but on the tuning and field steps it is a hard gate.

**`phase_coherence` — no echo, or a fake one.** This gate (used by
`tune.auto_phase` and `tune.rep_rate`, whose short flat traces defeat the
MAD-of-diff SNR estimate) checks that the trace shares one well-defined
phase. Its notes:

```text
[FAIL] phase_coherence: score=0.31 (min_r=0.7, n=4, structure_r=0.12, note=zero signal — no echo acquired)
```

The important one is the LO-leak note. A constant instrumental phasor (LO
leakage that survived the phase cycle, or a residual IQ offset) shares one
phase just as a real echo does, so a strongly-saturating `tune.rep_rate` curve
is additionally required to show real amplitude *structure* along that phase
axis. When the resultant is coherent but the deviations are structureless:

```text
[FAIL] phase_coherence: score=0.94 (min_r=0.7, n=24, structure_r=0.18, note=coherent resultant but structureless deviations — a constant offset (LO leakage), not an echo?)
```

means the "signal" is a constant offset, not an echo — check the phase cycle
and the IQ balance, not the sample.

**`relaxation_fit` — no believable decay.** The hard gate on `exp.t1` /
`exp.t2`. It scores the evidence (a per-point ΔAICc) that a relaxation signal
is present and described, so it passes a genuine decay at any noise level that
still holds a signal but rejects a flat or structureless trace:

```text
[FAIL] relaxation_fit: score=41 (min_delta_aicc_per_pt=0.375, delta_aicc_per_pt=0.1, n=200, adj_r2=0.12, rmse=0.03)
```

If the fit itself did not converge the note says so
(`note=fit failed: ...`) or `note=degenerate residuals (flat data?)`.

**`amplitude_rails` — π out of reach, and the fallback.** The fine amplitude
calibration fails on the rails when π cannot be reached within the sweep
(underpowered) or sits too low on the DAC scale (overpowered):

```text
[FAIL] amplitude_rails: score=99.5 (rail=high, high=99, hint=cannot reach pi: reduce vane attenuation)
[FAIL] amplitude_rails: score=28 (rail=low, low=30, hint=pi too low on the DAC scale: add vane attenuation)
```

A **rail** failure specifically triggers the runner's coarse-stage fallback:
if the protocol declared a `tune.power_for_length` step earlier, the runner
re-runs it (and the `tune.auto_phase` steps that followed it, which the vane
move invalidates) once and retries the calibration — so an underpowered sample
recovers without you. If there is no earlier coarse stage to fall back to, the
rail failure is reported as-is:

```text
      amplitude rail (high) hit, but the protocol has no earlier tune.power_for_length step — no fallback
```

and, if the fallback chain itself fails, the abort names the blocker:

```text
ABORTED: step tune.pi_calibration failed: [FAIL] amplitude_rails: ... (rail fallback blocked: field.edfs: [FAIL] echo_snr: ...)
```

**`temperature_band` — the setpoint never held.** `temp.wait` fails when the
band is not held within the wall-clock timeout:

```text
[FAIL] temperature_band: score=1800 (band_k=0.2, hold=3, timeout_s=1800, note=timeout before the band held)
```

The score is the elapsed seconds. Widen `band`, raise `timeout`, or check the
cryostat.

**`field.edfs` — flat versus weak.** When the field sweep finds no line above
the SNR floor, the `echo_snr` judge carries a `diagnosis` naming what to check.
The sweep first escalates once on its own (the span doubles and it re-runs):

```text
      no echo above the SNR floor — widening the span to +/- 500 G and re-running (one escalation)
```

If the second sweep also fails, one of two diagnoses surfaces — and the magnet
is **not** moved to a noise maximum:

```text
[FAIL] echo_snr: score=1.8 (min_snr=3, noise_sigma=..., diagnosis=weak line found near 3376.0 G (score 1.80) — increase scans, or narrow the range around it)
[FAIL] echo_snr: score=1.0 (min_snr=3, noise_sigma=..., diagnosis=flat everywhere: no echo anywhere in the range — check the resonator tuning, temperature, g-factor guess and sample position)
```

"weak line found" means something cleared the weak-line threshold — add scans
or narrow the range around the named field. "flat everywhere" means nothing
did — the advice points at the resonator, temperature, g-factor guess and
sample position, not at the scan count.

**Preset and calibration rejections** also surface as step failures (they are
`ValueError`s converted to `StepFailure`). The most common:

```text
<preset>: the DETECTION pulse does not move (st_inc == 0) — not a moving-echo (Hahn) T2 preset; exp.t2 needs the echo to follow the tau sweep
<preset>: mode 'amplitude' needs a 'Amplitude' preset, got 'Linear Time'
rep_rate: auto — no tune.rep_rate result in the session (run tune.rep_rate first)
apply_cal: no pi_calibration result in the session (run tune.pi_calibration first)
apply_cal: P3 would need 104.2% (> 100%) at 44.8 ns — re-run the coarse power stage (more power) or lengthen the pulse
```

## (c) Environment issues

These are not protocol bugs — they are about the machine the runner is talking
to.

**Param locks versus open control-center GUIs.** At the first real hardware
touch the runner seizes the `field.param` and `temp.param` cross-process
locks as `epr_auto` — the same discipline the experiment-runner GUIs use to
keep the interactive tools off the GPIB devices. If another tool already holds
one of those locks, the step fails and the run aborts, naming the holder:

```text
      step failed (attempt 1): temperature lock is held by 'temp_control' — another tool is driving the hardware; close it or wait
      step failed (attempt 1): field lock is held by 'awg_phasing_insys' — another tool is driving the hardware; close it or wait
```

The **temperature-control** window takes the `temp.param` lock while it is
open, so close it before a live run. The `field.param` lock is taken by
whichever tool is actively driving the magnet — an experiment-runner or the
CW/TR control window (`awg_phasing_insys`, `phasing_insys`, `cw_control`,
`tr_control`) — so stop that acquisition first. The interactive
**field-control** window itself never grabs the lock: it cooperatively backs
off, disabling its own setter and showing `Field control locked (epr auto
running)` while the runner owns the device, so it does not need closing. The
runner releases its own locks in a `finally` block and on `atexit`, so a crash
never leaves them stranded — and a dry-run never touches the lock files at
all.

**Run from the repository root.** The CLI changes into `libs/` itself, because
the Insys FPGA driver reads its `brd.ini` / `exam_adc.ini` relative to the
working directory at instantiation time. If it cannot find `libs/`:

```text
Cannot find the libs/ directory (looked at /home/.../libs); the Insys driver needs cwd=libs.
```

means you launched from the wrong place — run from the Atomize_ITC checkout
root.

**The GUI must be open for a live run.** A live acquisition worker pushes its
traces to the main window's LivePlot server; a real run started with no main
Atomize window dies at the first acquisition. (A dry-run needs no GUI — it
touches no hardware and no plot.) When the GUI *is* open, an engine run plots
into it exactly as an interactive scan does — the runner reaches the same
LivePlot socket, so you watch the sweep build live in the main window, and
`tune.echo_window` even shows the live "Dig" echo preview while it averages.
There is no separate window for the runner; its progress is the terminal log
plus the live plot in the GUI you already have open.

**Worker-side errors.** A crash inside the acquisition worker comes back as an
engine error and fails the step:

```text
      step failed (attempt 1): worker error:
      <traceback from the child process>
      step failed (attempt 1): worker exited without reporting a result
```

These are genuine hardware / driver faults (or a preset the pre-flight let
through that the live driver rejects); the child's traceback is the place to
look.

## (d) Warnings that are not errors

Some lines look alarming but are advisory — the run continues. They appear in
the `--test` pre-flight too, so you see them before the bench.

**Inert `target_snr`.** `target_snr` only ever lowers the scan count, so with
`scans` at its default of 1 there is nothing to shrink:

```text
      warning: target_snr 10 has no effect with scans: 1 — it only lowers the scan ceiling; set scans to the acceptable time budget
```

Set `scans` to the largest budget you are willing to spend and let
`target_snr` cut it short.

**Wrong preset family for `tune.rep_rate`.** Unlike the hard sweep-type checks
elsewhere, `tune.rep_rate` only warns when handed a Log Time or Amplitude
preset, then proceeds (and the coherence gate rejects the run if the metric
does cancel):

```text
      warning: 'Log Time' preset — its sweep flips the echo sign across the points, so the |mean| amplitude metric cancels; use a plain echo preset (Linear Time tau sweep) for rep_rate
```

**Canned-calibration skip (dry-run only).** In a dry-run the calibration is a
placeholder, so an over-rail scaling proves nothing about the real preset —
the slot is logged and left unpatched instead of failing:

```text
      apply_cal: P3 would need 104.2% (> 100%) with the canned dry-run calibration — left unpatched
```

**Deliberate `apply_cal: none`.** Opting a step out of patching, with a
calibration in the session, is announced so it is never a silent no-op:

```text
      apply_cal: none — pi_calibration deliberately not applied, preset-stored values in force
```

The next two are **load-time** advisories: they are computed while the protocol
parses, so they print once at the top of the run (right after the `=== … ===`
header, before the first step) and in the `epr-auto validate` output as well as
the `--test` dry-run.

**A shipped default preset was edited.** The AWG phasing GUI saves into the same
`experiments/` folder that holds the shipped defaults, so a GUI save can
overwrite a preset automation falls back to. When a default's bytes no longer
match the recorded fingerprint, the runner warns but still runs the modified
file:

```text
      warning: default preset 'hahn_echo_4s.phase_awg' differs from the shipped version (edited in the GUI?) — the run will use the modified file; restore it with git, or save custom presets into atomize/epr_auto/presets/ under a new name
```

Restore the original with `git checkout -- <preset>`, or move your edit into the
user preset directory (`atomize/epr_auto/presets/`) under a new name and
reference that name from the protocol (see [Presets](presets.md)). If you
changed a default on purpose, re-record the fingerprints with
`python3 -m atomize.epr_auto.preset_hash --update`.

**Tuning before the field is set.** A `tune.auto_phase` or `tune.pi_calibration`
that runs before any `field.*` step has appeared is flagged, because on a cold
start there may be no echo to tune on until the magnet is parked on the line:

```text
      warning: tune.auto_phase (line 13) runs before any field.* step — on a cold start there may be no echo to phase on; make sure the magnet is already parked on the line
```

This is legitimate when the magnet is already sitting on the resonance (a
warm-start re-tune, or a manually pre-set field), which is why it is only a
warning. If it is not intentional, put a `field.edfs` or `field.set` step ahead
of the tuning.

## Next steps

- [Writing protocols](protocols.md) — the schema whose validators produce the
  load-time errors above.
- [The tune-up chain](tuning.md) — the judges and invalidation rules behind
  the step failures.
- [Presets](presets.md) — the demodulated 1-D mode and the sweep-type
  families.
