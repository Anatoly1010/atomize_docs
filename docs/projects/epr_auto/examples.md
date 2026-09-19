# Examples

The `protocols/` directory in the Atomize_ITC checkout contains examples for a single T2, a field series, temperature series and preliminary tuning. The temperature examples below keep confirmed T1/T2 curves and use their measured baseline to prepare the next temperature's range. Read [Writing protocols](protocols.md) for YAML syntax and [The tune-up chain](tuning.md) for calibration order.

## A single T2 with automatic 2τ range — `t2_auto_range.yaml`

[`t2_auto_range.yaml`](https://github.com/Anatoly1010/Atomize_ITC/blob/main/protocols/t2_auto_range.yaml) measures one T2 at the current temperature and checks its time range along the physical evolution axis 2τ. It sets the selected field, phases the signal and calibrates the pulse amplitudes before the decay measurement. `adjust_range: true` checks the measured tail during the first one to three complete scans. A sufficient range continues accumulating toward SNR 20 in the same acquisition; excess baseline is retained. A clearly short range can receive one early extension within the shared 600 s budget. The 64 scans are a ceiling, and the requested SNR is not guaranteed.

```yaml
# Set the sample, field and initial T2 range for the experiment.
sample: t2_auto_range
autonomy: checkpointed
notify: none

steps:
  - field.set:
      value: 3318 G
  - tune.auto_phase
  - tune.pi_calibration:
      mode: amplitude
  - exp.t2:
      preset: hahn_echo_4s.phase_awg
      tau_start: 300 ns
      tau_step: 20 ns
      points: 400
      scans: 64
      target_snr: 20
      max_duration: 600 s
      adjust_range: true
```

Set `sample`, the 3318 G field and the initial `tau_start`, `tau_step` and `points` for the experiment. The saved time axis is 2τ, so its increment is twice the hardware-rounded `tau_step`. This example uses the shipped Hahn-echo and amplitude-calibration presets; its repetition rate and echo integration window come from the presets. It does not change temperature or run `tune.echo_window`. Set `adjust_range: false` to retain the initial time range.

```bash
python3 -m atomize.epr_auto validate protocols/t2_auto_range.yaml
python3 -m atomize.epr_auto run protocols/t2_auto_range.yaml --test
```

## A single T2 — `overnight_t2.yaml`

```yaml
sample: test_sample
autonomy: checkpointed        # supervised | checkpointed | autonomous

steps:
  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
      checkpoint: true        # confirm the chosen field before continuing

  - tune.auto_phase           # phase on the echo at the working field

  - tune.pi_calibration:
      mode: amplitude         # fine stage; amplitude sweep at fixed length

  - exp.t2:
      tau_start: 300 ns
      tau_step: 12 ns
      points: 400
      scans: 16
```

**`field.edfs`** sweeps the magnet across 338–352 mT, picks the working field
at the magnitude maximum of the echo-detected sweep, and parks the magnet
there — storing the field for every later step. It is marked
`checkpoint: true`: in `checkpointed` autonomy the runner pauses here for the
operator to confirm the chosen field before the magnet moves (in a dry-run the
checkpoint is logged and auto-continued). The stored field flows into the
build of every acquisition after it.

**`tune.auto_phase`** runs after `field.edfs` so there is an echo at the
working field to phase on. It acquires a short echo on the default
`hahn_echo_4s.phase_awg` preset and zeroes the receiver phase, storing the
corrected zero-order in the session. This protocol carries no
`tune.echo_window` before it, so the phase is measured over the preset's own
stored integration window — the simplest case; a fuller tune-up measures the
window first (see [The tune-up chain](tuning.md)).

**`tune.pi_calibration`** runs the fine amplitude calibration on the default
`ampl_4s.phase_awg` Amplitude preset, fitting the π and π/2 AWG amplitudes
independently and storing them. Because the protocol declares no
`tune.power_for_length` coarse stage before it, there is no rail fallback to
fall back on: if the fit runs into the amplitude rails the step fails as-is
(see [Troubleshooting](troubleshooting.md)).

**`exp.t2`** measures the Hahn-echo decay. The session state assembled by the
three tuning steps flows into its build automatically: the zeroed phase from
`tune.auto_phase`, the working field from `field.edfs`, and — through
`apply_cal`, inferred from the preset's own two amplitude levels — the
calibrated π and π/2 amplitudes from `tune.pi_calibration`. The `tau_start` /
`tau_step` re-anchor the tau sweep, and the saved axis is the physical
evolution time `2·tau`. `scans: 16` is a **fixed** budget here: this protocol
sets no `target_snr` and no `max_duration`, so all 16 scans always run. That
is the right choice when you know the scan count you want; the next example
shows how to make the runner decide it.

### What a live run leaves behind

Run live (not `--test`), this protocol writes into
`~/epr_data/epr_auto_<date>_test_sample/`:

- **`manifest.json`** — the run record, rewritten after every step. It holds
  the protocol name, sample, autonomy, start/finish timestamps and status,
  and one entry per step with its resolved parameters, result, judge reports
  and attempt count. The `exp.t2` entry, for instance, carries the fitted
  `t2`, the stretched-exponential `beta`, and the `echo_snr` /
  `relaxation_fit` judge scores.
- **`protocol_overnight_t2.yaml`** — a verbatim copy of the protocol, so the
  exact YAML that produced the data always sits next to it.
- **the acquisition CSVs** — `001_edfs.csv`, `002_auto_phase.csv`,
  `003_pi_cal_amplitude.csv`, `004_t2.csv`, each three columns
  (axis, I, Q). A dry-run writes none of this; it logs the same information
  to the terminal.

## A field series — `field_series_t1t2.yaml`

```yaml
sample: field_series
autonomy: checkpointed

steps:
  # Tune once at the line maximum; field moves do NOT invalidate the phase,
  #  so the loop reuses this tune.
  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
      checkpoint: true
  - tune.echo_window
  - tune.auto_phase
  - tune.pi_calibration:
      mode: amplitude

  # One T2 + T1 per field. A dead position records + continues to the next.
  - foreach:
      var: B
      values: ['3000 G', '3318 G', '3376 G', '3450 G']
      on_fail: continue
      steps:
        - field.set:
            value: $B
        - exp.t2:
            tau_start: 300 ns
            tau_step: 12 ns
            points: 200
            scans: 48
            target_snr: 10
        - exp.t1:
            t_start: 500 ns
            t_end: 2 ms
            points: 200
            scans: 48
            target_snr: 10
            rep_rate: 100
```

### Tune once, at the top

The four steps before the `foreach` are a complete tune-up performed **once**,
at the line maximum found by the first `field.edfs`: find the line, measure
the integration window, zero the phase, calibrate the pulses. Here
`tune.echo_window` runs before `tune.auto_phase` — the canonical order, since
auto-phase integrates over the window the previous step measured. The window,
phase, and calibration land in the session and are reused for every field in
the loop.

The key design fact is in the comment: **the series does not re-phase on
field moves.** Moving the magnet changes neither the resonator tuning nor B₁,
and its effect on T₁ is minor, so a field move drops no calibration — the
phase, window, field-independent calibration, and rep-rate recommendation all
survive it. Measured on real hardware, the phase drifted only ±1–3.5° across
the whole line; a few degrees of demod drift costs nothing, because every
relaxation curve is re-rotated onto its principal axis before fitting.
Temperature moves are the opposite — they *do* force a
re-phase — which is why a *temperature* series, unlike this field series,
follows each move with a fresh `tune.auto_phase`. See
[The tune-up chain](tuning.md#calibration-flow-and-invalidation) for the full
invalidation table.

### The `foreach` block

`foreach` runs its sub-steps once for each value of `B`, substituting the
value in wherever `$B` appears. Here `field.set: {value: $B}` moves the magnet
to each field in turn, then `exp.t2` and `exp.t1` measure the two relaxation
curves at that field. The four values are given as **quoted field strings**
(`'3000 G'`, …) because `value:` needs a `"<value> <unit>"` string; every
substituted sub-step is fully validated for every value at load time, so a
typo is caught by `epr-auto validate` before the run.

Two `foreach` behaviours earn their keep here:

- **`on_fail: continue`** — a dead field position (no echo, a failed fit) is
  recorded in the manifest and the series moves on to the next value rather
  than aborting the whole night. A per-iteration failure that is an explicit
  operator abort or an unexpected code error is *not* swallowed, but an
  ordinary bad-data failure is.
- **loop tagging** — each acquisition's CSV carries the loop stamp, so the
  files are self-identifying: `005_t2_B_3318G.csv` is a T2 at the `B = 3318 G`
  point. The manifest additionally records `{var: B, value: '3318 G',
  index: 2}` on every step run inside the block, tying each acquisition back
  to its position in the series.

The rail-triggered coarse fallback deliberately does **not** reach inside the
`foreach` — the intent is to tune once before the loop, so a sub-step's
amplitude-rail failure is handled by the block's `on_fail` rather than by
re-running an earlier coarse stage.

### The scans ceiling and `target_snr`

Both experiments set `scans: 48` **and** `target_snr: 10`. This is the pairing
that makes an unattended field series efficient. `scans` is a **ceiling**, not
a fixed count: after each completed scan the runner measures the accumulated
curve's SNR with the same `echo_snr` judge that gates the finished step, and
stops as soon as the curve reaches SNR 10 — projecting via √N scaling whether
the target is even reachable inside the ceiling. At the line maximum (3318 /
3376 G) a curve clears SNR 10 in a handful of scans; on the weak 3000 G
shoulder it runs the full 48. This is exactly the adaptation an operator
does by hand — a few scans at the line maximum versus dozens on a weak
shoulder — now automatic.

`target_snr` only ever *lowers* the scan count; it never adds scans to chase
the target. That is why it is paired with a real ceiling: with `scans` at its
default of 1 there is nothing to shrink, and the step would warn that the
setting is inert. Set `scans` to the largest budget you are willing to spend
per point, and let `target_snr` cut each easy point short.

`exp.t1` also fixes `rep_rate: 100` (Hz) rather than inheriting the preset's
value — a deliberate choice for a quantitative recovery measurement. It could
instead be `rep_rate: auto` if a `tune.rep_rate` step had run in the prologue
to measure and recommend a rate; see
[rep_rate: auto](protocols.md#rep_rate-auto).

!!! note
    A field series does not need `max_duration`, but an open-ended overnight
    run does: adding `max_duration: 21600 s` to an experiment step caps its
    wall-clock time, shrinking the scan count mid-run to finish inside the
    budget (the data acquired so far is always kept). When both
    `target_snr` and `max_duration` are set the smaller resulting scan count
    wins.

## A temperature series — T1 and T2

[`temperature_series_t1t2.yaml`](https://github.com/Anatoly1010/Atomize_ITC/blob/main/protocols/temperature_series_t1t2.yaml) sets the field to an editable 3318 G, reaches 80 K, and measures the echo window, phase and fine pulse calibration once at that starting temperature. Its `foreach` then sets and waits at 80, 100, 120 and 140 K, refreshes the echo window and phase, and acquires T2 and T1 with `adjust_range: true`. The initial T2 seed is 300 ns plus 400 points at a 20 ns tau step; T1 begins at 500 ns and ends at 5 ms with an explicit 100 Hz rate.

Each experiment requests `target_snr: 20`, with `scans: 64` as a ceiling and a projected 600 s budget shared between its initial and any extended acquisition. The range is checked after the first full scan, with up to three scans when noisy. A suitable range keeps accumulating in the same acquisition; a clearly unfinished tail can trigger one early extension before spending the full SNR budget. If the early check remains uncertain, accumulation continues without a late repeat. For a carried or repaired T1 range, the runner recomputes the maximum timing-compatible rate; on Nd:YAG this is fixed at 9.9 Hz and the step fails if the sequence does not fit. Scan/time limits and an optimistic SNR projection can leave the final SNR below the target.

The accepted measured curve can guide the next temperature's range; the current curve with a confirmed plateau is kept even if it has excess baseline. The fine calibration remains valid across temperature moves while the phase and window are refreshed.

```yaml
# Set the sample, field, and shipped presets for the real setup before acquisition.
sample: temperature_series_t1t2
autonomy: checkpointed
notify: none

steps:
  - field.set:
      value: 3318 G
  - temp.set:
      setpoint: 80
      heater_range: 5 W
  - temp.wait:
      band: 0.3
      timeout: 1800 s
  - tune.echo_window
  - tune.auto_phase
  - tune.pi_calibration:
      mode: amplitude

  - foreach:
      var: T
      values: [80, 100, 120, 140]
      on_fail: continue
      steps:
        - temp.set:
            setpoint: $T
            heater_range: 5 W
        - temp.wait:
            band: 0.3
            timeout: 1800 s
        - tune.echo_window
        - tune.auto_phase
        # Seed the first curve; later temperatures reuse accepted measured ranges.
        - exp.t2:
            tau_start: 300 ns
            tau_step: 20 ns
            points: 400
            scans: 64
            target_snr: 20
            max_duration: 600 s
            adjust_range: true
        - exp.t1:
            t_start: 500 ns
            t_end: 5 ms
            points: 200
            scans: 64
            target_snr: 20
            max_duration: 600 s
            rep_rate: 100
            adjust_range: true
```

```bash
python3 -m atomize.epr_auto validate protocols/temperature_series_t1t2.yaml
python3 -m atomize.epr_auto run protocols/temperature_series_t1t2.yaml --test
```

## A temperature series — T2 only

[`temperature_series_t2.yaml`](https://github.com/Anatoly1010/Atomize_ITC/blob/main/protocols/temperature_series_t2.yaml) uses the same starting-temperature calibration and temperature loop. It adds a quantitative `tune.rep_rate` at every temperature before `exp.t2` with `rep_rate: auto`, because temperature moves make the earlier rate recommendation stale. It also combines `target_snr: 20`, `scans: 64` and `max_duration: 600 s` with the same early range checks. The range memory carries only the sampled range, never the old rate.

```yaml
# Set the sample, field, and shipped presets for the real setup before acquisition.
sample: temperature_series_t2
autonomy: checkpointed
notify: none

steps:
  - field.set:
      value: 3318 G
  - temp.set:
      setpoint: 80
      heater_range: 5 W
  - temp.wait:
      band: 0.3
      timeout: 1800 s
  - tune.echo_window
  - tune.auto_phase
  - tune.pi_calibration:
      mode: amplitude

  - foreach:
      var: T
      values: [80, 100, 120, 140]
      on_fail: continue
      steps:
        - temp.set:
            setpoint: $T
            heater_range: 5 W
        - temp.wait:
            band: 0.3
            timeout: 1800 s
        - tune.echo_window
        - tune.auto_phase
        - tune.rep_rate:
            mode: quantitative
            rate_min: 10
            rate_max: 2000
        # Seed the first curve; later temperatures reuse accepted measured ranges.
        - exp.t2:
            tau_start: 300 ns
            tau_step: 20 ns
            points: 400
            scans: 64
            target_snr: 20
            max_duration: 600 s
            rep_rate: auto
            adjust_range: true
```

```bash
python3 -m atomize.epr_auto validate protocols/temperature_series_t2.yaml
python3 -m atomize.epr_auto run protocols/temperature_series_t2.yaml --test
```

Run these commands from the Atomize_ITC checkout. Set `sample`, the field, both initial and loop temperatures, heater range, pulse presets, target SNR, scan ceilings and initial time ranges for the experiment. The initial range should be long enough to include a plateau at the first temperature; it is used again when a new run starts. Omitted presets select the shipped defaults; add `preset:` to the tuning and experiment steps to use your own matching sequences. `on_fail: continue` records a failed temperature point and proceeds to the next temperature. After checking the dry-run, omit `--test` for acquisition. Files and `manifest.json` are saved under `~/epr_data/epr_auto_<date>_<sample>` unless `output` is set. Both dry-runs complete with canned data; they check protocol wiring and device test paths, not the measured plateau or live range carryover. [Adaptive relaxation ranges](protocols.md#adaptive-relaxation-ranges) explains the acceptance and extension rules.

## Next steps

- [Writing protocols](protocols.md) — the full YAML schema behind these
  files.
- [Troubleshooting](troubleshooting.md) — what the failures these protocols
  can hit mean, verbatim.

## A live repetition-rate check — `rep_rate_live.yaml`

[`rep_rate_live.yaml`](https://github.com/Anatoly1010/Atomize_ITC/blob/main/protocols/rep_rate_live.yaml) fixes the field at 3318 G, runs `tune.auto_phase`, and keeps one digitizer card open while scanning 10–2000 Hz. Each rate accepts three fresh nonempty complex echo curves within 5%; `scans: 1` requests one disjoint stable group. The protocol uses a 120 s timeout per rate and quantitative fitting. Set the sample and field for your experiment. The protocol passes test mode; live transitions still need validation on the spectrometer.

```yaml
steps:
  - field.set: {value: 3318 G}
  - tune.auto_phase
  - tune.rep_rate:
      rate_min: 10
      rate_max: 2000
      steps: 6
      points: 3
      scans: 1
      max_wait: 120 s
      mode: quantitative
```

## Preliminary tuning and handoff

This example reproduces `protocols/preliminary_tuning.yaml`. Set the sample, scan bounds, RV attenuation and pulse length before use. The built-in ringing and resonator steps share the IF of the later echo preset and take no external preset. For resonator selection, `window: 4 ns` is recommended; the shipped example leaves it at the `2 ns` default.

```yaml
# Dry-run: python3 -m atomize.epr_auto run protocols/preliminary_tuning.yaml --test
sample: test_sample
autonomy: supervised
notify: none

# Set the sample, synthesizer bounds and field range for the experiment.
steps:
  # Built-in sequences; both IF values must match the later echo preset.
  - tune.ringing_check:
      if_mhz: 50
      pulse_length: 102.4 ns
  # Optional:
  #   done: true
  # Optional: omit this step to keep the current synthesizer frequency.
  - tune.resonator:
      if_mhz: 50
      start_mhz: 9200
      end_mhz: 9600
      step_mhz: 1
  # Optional: an absolute synthesizer frequency instead of the resonator scan.
  # - bridge.set:
  #     frequency_mhz: 9440
  - tune.find_echo:
      preset: hahn_echo_4s.phase_awg
      center: 3445 G
      span: 100 G
      attenuation_db: 10
      # Positive shifts the echo frequency above the resonator center.
      frequency_shift_mhz: 0
      # Target pi length; every echo pulse takes it, pi differs from pi/2 only in amplitude.
      pulse_length: 22.4 ns
      points: 41
  - tune.maximize_echo:
      preset: hahn_echo_4s.phase_awg
      # pi/2 amplitude a in %, pi at 2a; the optimum must lie inside the range.
      amplitude_range: [5, 50]
      coarse_step: 5
      fine_step: 1
      field_span: 10 G
      points: 21
      pulse_map: {P2: pi2, P3: pi}
  # Writes preset copies and fine_tuning.yaml into the run's handoff directory.
  - tune.save_presets:
      preset: hahn_echo_4s.phase_awg
      calibration_preset: ampl_4s.phase_awg
      field_preset: ed_4s.phase_awg
      # Rabi pulse length for the fine calibration; omitted, so the preliminary length is used.
      # calibration_length: 16 ns
```

Use a signed `frequency_shift_mhz`, such as `-50`, to optimize the echo below the resonator center in a two-frequency experiment. The live final step publishes four presets and `fine_tuning.yaml` to `tuned/` beside the protocol, with an archive copy in the run directory. `calibration_length` is omitted here, so the Rabi pulse and both echo pulses in the exported field and calibrated-echo presets use the preliminary `22.4 ns` length. The generated EDFS uses the 100 G search span, recentered on the tuned field, and 200 points; the 10 G span is only the preliminary field refinement. Test mode reports the handoff without writing files. See [Preliminary tuning](tuning.md#preliminary-tuning) for the phase, ringing-check and RV-settling requirements.
