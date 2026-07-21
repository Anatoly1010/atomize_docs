# Examples

Two shipped protocols, walked through step by step. Both live in the
`protocols/` directory of the Atomize_ITC checkout and both dry-run with no
hardware and no GUI:

```bash
epr-auto run protocols/overnight_t2.yaml --test
epr-auto run protocols/field_series_t1t2.yaml --test
```

The first is the smallest useful protocol — tune, then measure one T2 with a
fixed scan budget. The second is the pattern most real campaigns follow —
tune once, then repeat a measurement group across a series of fields, letting
each position spend only the scans it needs. Read
[Writing protocols](protocols.md) for the YAML dialect and
[The tune-up chain](tuning.md) for the physics behind the tuning steps; this
page is about how the pieces fit together in a working file.

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — both protocols are implemented
    and verified in dry-run (`--test`) only. Always dry-run first, and keep
    the first live sessions in `supervised` autonomy with an operator
    present.

## A single T2 — `overnight_t2.yaml`

```yaml
sample: test_sample
autonomy: checkpointed        # supervised | checkpointed | autonomous

steps:
  - tune.auto_phase

  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
      checkpoint: true        # confirm the chosen field before continuing

  - tune.pi_calibration:
      mode: amplitude         # fine stage; amplitude sweep at fixed length

  - exp.t2:
      tau_start: 300 ns
      tau_step: 12 ns
      points: 400
      scans: 16
```

**`tune.auto_phase`** acquires a short echo on the default
`hahn_echo_4s.phase_awg` preset and zeroes the receiver phase, storing the
corrected zero-order in the session. This protocol carries no
`tune.echo_window` before it, so the phase is measured over the preset's own
stored integration window — the simplest case; a fuller tune-up measures the
window first (see [The tune-up chain](tuning.md)). Note the ordering: the
phase is measured *before* the field is set. That is deliberate and safe —
the phase is field-independent (measured on the 2026-07 oTP campaign), so the
field move in the next step does not invalidate it.

**`field.edfs`** sweeps the magnet across 338–352 mT, picks the working field
at the magnitude maximum of the echo-detected sweep, and parks the magnet
there — storing the field for every later step. It is marked
`checkpoint: true`: in `checkpointed` autonomy the runner pauses here for the
operator to confirm the chosen field before the magnet moves (in a dry-run the
checkpoint is logged and auto-continued). The stored field flows into the
build of every acquisition after it.

**`tune.pi_calibration`** runs the fine amplitude calibration on the default
`ampl_4s.phase_awg` Amplitude preset, fitting the π and π₂ AWG amplitudes
independently and storing them. Because the protocol declares no
`tune.power_for_length` coarse stage before it, there is no rail fallback to
fall back on: if the fit runs into the amplitude rails the step fails as-is
(see [Troubleshooting](troubleshooting.md)).

**`exp.t2`** measures the Hahn-echo decay. The session state assembled by the
three tuning steps flows into its build automatically: the zeroed phase from
`tune.auto_phase`, the working field from `field.edfs`, and — through
`apply_cal`, inferred from the preset's own two amplitude levels — the
calibrated π / π₂ amplitudes from `tune.pi_calibration`. The `tau_start` /
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
- **the acquisition CSVs** — `001_auto_phase.csv`, `002_edfs.csv`,
  `003_pi_cal_amplitude.csv`, `004_t2.csv`, each three columns
  (axis, I, Q). A dry-run writes none of this; it logs the same information
  to the terminal.

## A field series — `field_series_t1t2.yaml`

```yaml
sample: field_series
autonomy: checkpointed

steps:
  # Tune once at the line maximum; field moves do NOT invalidate the phase
  # (measured on the 2026-07 oTP campaign) so the loop reuses this tune.
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
survive it. This was measured directly on the 2026-07 oTP campaign (phase
drifted only ±1–3.5° across the whole line; a few degrees of demod drift costs
nothing, because every relaxation curve is re-rotated onto its principal axis
before fitting). Temperature moves are the opposite — they *do* force a
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
shoulder it runs the full 48. This is exactly the adaptation the campaign
operator did by hand — 6 scans at the line max versus 32–46 at the shoulder —
now automatic.

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

## Next steps

- [Writing protocols](protocols.md) — the full YAML schema behind these
  files.
- [Troubleshooting](troubleshooting.md) — what the failures these protocols
  can hit mean, verbatim.
