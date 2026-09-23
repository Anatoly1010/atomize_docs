# Writing protocols

A protocol is a single YAML file that lists the steps `epr_auto` runs, in
order, together with the autonomy level and a few run-wide settings. This
page is the reference for that YAML dialect: the top-level keys, how a step
is written, the value syntax every parameter shares, the per-step failure
and checkpoint controls, and the `foreach` series block. For the parameters
of each individual step see the [Step reference](steps.md); for the tuning
chain the steps compose into, see [The tune-up chain](tuning.md); for what a
step expects of the preset it names, see [Presets](presets.md). Two complete
protocols are annotated on the [Examples](examples.md) page.

Validate a file at any time without running it:

```bash
epr-auto validate my_protocol.yaml
```

Validation is thorough: it resolves every preset path, range-checks every
parameter, expands `foreach` blocks, and reports the first problem as
`INVALID: <file>:<line>: <message>`. A protocol that validates will load; it
does not guarantee the hardware will cooperate.

## Top-level keys

The document is a YAML mapping with exactly these keys; any other top-level
key is a load-time error.

| Key | Required | Value | Purpose |
| --- | --- | --- | --- |
| `sample` | yes | non-empty string | sample name; used in the run-directory name and the manifest |
| `steps` | yes | non-empty list | the ordered steps (and `foreach` blocks) to run |
| `autonomy` | no | `supervised` \| `checkpointed` \| `autonomous` | how much the runner pauses for the operator (default `supervised`) |
| `output` | no | run-directory template string | where acquisitions and the manifest are written |
| `notify` | no | `none` \| `telegram` | operator notifications (default `none`) |

A minimal but complete protocol therefore needs only `sample` and `steps`:

```yaml
sample: test_sample
autonomy: checkpointed

steps:
  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
  - tune.auto_phase
  - exp.t2:
      tau_start: 300 ns
      tau_step: 12 ns
      points: 400
      scans: 16
```

### output — the run-directory template

Each run writes its CSV acquisitions and a `manifest.json` into a run
directory. Without `output`, that directory is
`~/epr_data/epr_auto_<date>_<sample>`. When `output` is given it is a
template string in which only two placeholders expand: `{date}` (today's
date, ISO `YYYY-MM-DD`) and `{sample}` (the sample name, with unsafe
characters replaced). A leading `~` expands to your home directory.

```yaml
output: ~/epr_data/{date}_{sample}
```

A leading `~` expands to your home directory; an absolute path is used as
given. A **relative** template resolves against the directory you launched
`epr-auto` from — not `libs/`, which the CLI has already `chdir`'d into by the
time the first save happens — so `output: runs/{sample}` lands under your
working directory as you would expect.

If the target directory already holds a `manifest.json` — a same-day re-run
of the same sample — the runner appends a `_run2` / `_run3` … suffix and logs
the choice, so a repeat run never overwrites an earlier one's manifest and
low-numbered CSVs.

The template is checked at load time, not at run start, so a typo such as an
unsupported placeholder fails `epr-auto validate` immediately rather than
only when a real run reaches its first save:

```text
INVALID: my_protocol.yaml:4: 'output' template: only {date} and {sample}
placeholders are supported ('run')
```

### notify

`notify: telegram` sends operator notifications — checkpoints
auto-approved in autonomous mode, skipped and failed steps, and the
run's finish or abort — through
[`general.bot_message`](../../functions/general_functions/general_functions.md),
which needs a bot token and chat id in `main_config.ini` (see the
[configuration section of the usage page](../../usage.md)). The default
`none` logs the same messages to the terminal only. Notifications never fire in a dry-run and a
notification failure never takes down a run.

## Preliminary receiver control

Video attenuation (VA) sets the receiver signal level sent to the ADC. The bridge has two video attenuators: Video Attenuation 1 (VA1, `video1_db`) and Video Attenuation 2 (VA2, `video2_db`). The rotary-vane attenuator (RV, `attenuation_db`) sets microwave excitation power at the sample.

`tune.find_echo` enables video-attenuation adjustment by default. Set `adjust_video: false` to retain the current VA settings; `tune.maximize_echo` and `tune.video_attenuation` inherit this choice unless they explicitly override it. When enabled, the RV approach uses live receiver monitoring and a 200 mV threshold. See [Preliminary tuning](tuning.md#preliminary-tuning) for the approach and recovery sequence.

`rep_rate` on the preliminary echo search and maximization accepts a number between 0.1 and 10000 Hz, or `auto` from an earlier accepted `tune.rep_rate` result. The resolved automatic rate must also be within this range. `tune.rep_rate` uses a fixed field/fixed tau and ordinary nonempty `digitizer_get_curve(live_mode=1)` results. Curves labelled with an earlier rate step or a repeated buffer are discarded; mixed-rate content within one curve is left to the stability check. Each curve holds one complete phase cycle, which may span several ADC buffers. Its tuning grid has a 10 Hz lower bound and a 10 Hz default `rate_min`; ordinary rates and recommendations retain the 0.1 Hz hardware floor. `points` is the minimum 3-curve 5% stability window and `scans` counts disjoint stable groups. During tuning the Worker pins a 512 KB ADC buffer regardless of the ADC window or rate, then restores the previous setting after the card closes, including on Stop or failure. `max_wait` is a per-rate timeout including buffer arrival. `bridge.set` accepts `video1_db` from 0 to 30 in 2 dB steps and `video2_db` from 0 to 31.5 in 0.5 dB steps; values between hardware settings are rejected. At least one RV, synthesizer or video setting is required.

## Steps

`steps` is an ordered list. Each entry is either an ordinary step or a
`foreach` block (described below). A step may be written two ways.

A step that needs no parameters is a bare string:

```yaml
steps:
  - tune.echo_window
  - tune.auto_phase
```

A step that takes parameters is a single-key mapping — the step name, then
its parameters indented beneath it:

```yaml
steps:
  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
```

The most common structural mistake is under-indenting the parameters so YAML
reads them as sibling list entries; the runner reports that as
`each step must be a single "name: {params}" mapping (check the indentation
of the parameters)`. An unknown step name, or an unknown parameter for a
known step, is likewise a load-time error that names the valid
alternatives.

Order matters physically, and one ordering slip is caught for you: a
`tune.auto_phase`, `tune.pi_calibration` or `tune.power_for_length` placed
before any `field.*` step raises a load-time **warning** (not an error — tuning
at a manually pre-set field is legitimate), because on a cold start there is no
echo to tune on until the magnet is on the line. The canonical tune-up therefore sets the field
(`field.edfs`, or `field.set`) before it phases and calibrates; see
[The tune-up chain](tuning.md). The warning is described under
[Warnings that are not errors](troubleshooting.md#d-warnings-that-are-not-errors).

## Parameter value syntax

Parameters share the framework-wide value conventions.

**Times and fields are `"<value> <unit>"` strings** (YAML plain scalars —
no quotes needed outside flow lists). Time units are
`ps`, `ns`, `us`, `ms`, `s`, `ks`; field units are `G`, `mT`, `T`. A bare
number where a unit string is expected is rejected.

```yaml
      tau_start: 300 ns
      t_end: 5 ms
      range: [338 mT, 352 mT]
      value: 3318 G
```

**Plain numbers** are used where the quantity has a fixed, implied unit —
`scans: 16` (a count), `amplitude: 95` (AWG percent), `setpoint: 80.0`
(kelvin), `rep_rate: 100` (Hz), `g: 2.0023`.

**`auto`** is a literal keyword accepted by a few parameters in place of an explicit value: `range: auto` on `field.edfs` centres the sweep on the resonance computed from the synthesizer readout, and `rep_rate: auto` on preliminary echo tuning and experiment steps pulls in the `tune.rep_rate` recommendation (see below).

**Mappings** are used where a parameter carries structured data. The clearest
example is `apply_cal` on the experiment steps, a pulse-slot-to-role map:

```yaml
  - exp.t2:
      points: 400
      apply_cal: {P2: pi2, P3: pi}
```

`apply_cal` also accepts the literal `none` to deliberately skip patching the
preset with the fine calibration; omitting it entirely infers the map from
the preset's own amplitude levels. Slots are `P2` through `P9` and roles are
`pi` or `pi2`.

## Per-step control keys

Alongside a step's own parameters, three keys control how the runner treats
that step. They are valid on any step and are stripped before the step's
parameters are validated.

| Key | Value | Default | Effect |
| --- | --- | --- | --- |
| `retries` | integer ≥ 0 | `0` | extra attempts after the first failure, before `on_fail` applies |
| `on_fail` | `abort` \| `skip` \| `ask` | `abort` | what to do once all attempts are exhausted |
| `checkpoint` | `true` \| `false` | `false` | pause for operator confirmation before this step (in `checkpointed` mode) |

```yaml
  - tune.pi_calibration:
      mode: amplitude
      retries: 1
      on_fail: ask
```

`on_fail` decides the fate of a step that still fails after its retries are
spent:

- **`abort`** (default) stops the run. The manifest records the step as
  `failed` and the run status as aborted.
- **`skip`** continues the protocol without the step; the manifest records
  it as `failed-skipped` and later steps that depend on its result run with
  whatever the session already holds.
- **`ask`** prompts the operator in the launcher dialog or terminal to retry, skip, or abort.
  This needs an attended run: in an ordinary CLI dry-run, in
  `autonomous` mode, or without either GUI interaction or a terminal, `ask` degrades to `abort` (and notifies
  that it did so), because there is no one to answer.

`checkpoint: true` marks a step the operator should confirm before it runs —
typically one that moves the vane or sets the field. Whether the checkpoint
actually pauses depends on the autonomy level.

## Autonomy levels

`autonomy` sets how often the runner stops for a human.

| Level | Pauses before |
| --- | --- |
| `supervised` | every step |
| `checkpointed` | only steps marked `checkpoint: true` |
| `autonomous` | nothing — runs unattended end to end |

In `autonomous` mode a `checkpoint: true` step is auto-approved with a
notification rather than a pause, so an overnight run is never left waiting
on a prompt. The judges (see below) remain the only brake on data quality.

A checkpoint that would pause but has neither a launcher dialog nor a terminal to prompt at — an
unattended run in `supervised` or `checkpointed` mode — is a hard abort, not
a silent continue, so a batch job cannot slip past a confirmation the author
demanded. In an ordinary CLI dry-run every checkpoint and every operator prompt is
auto-continued and logged, so `--test` exercises the full step list without
stopping.

Preliminary-tuning safety failures are hard aborts: ringing above a threshold, invalid traces and other preliminary acquisition failures stop the run and attempt to return RV to 60 dB. Step retries, `on_fail: skip` and `foreach` continuation do not override them. See [Preliminary tuning](tuning.md#preliminary-tuning).

## Series: the foreach block

A `foreach` block runs its sub-steps once for each value of a loop variable,
substituting the value into the sub-steps. It is how a field series or a
temperature series is written: tune once, then repeat a measurement group
across a list of positions. A worked field series is annotated on the
[Examples](examples.md) page.

```yaml
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

A `foreach` mapping takes exactly `var`, `values`, `steps`, and the optional
`on_fail`; any other key is a load-time error, and a `foreach` cannot be
nested inside another.

- **`var`** is the loop-variable name — letters, digits, and underscores,
  not starting with a digit.
- **`values`** is a non-empty list of the values to iterate over.
- **`steps`** is the non-empty sub-step list, written exactly like the
  top-level `steps`.
- **`on_fail`** is `continue` (default) or `abort` — the per-iteration
  failure policy, described below.

### Substitution rules

Inside the block, `$var` in a sub-step is replaced with the current value.
Substitution follows precise rules, and every substituted sub-step is fully
parsed and validated for every value at load time, so a typo in a
substituted parameter is caught before the run rather than partway through
the series.

- **Whole names only.** `$B` matches the variable `B` but never fires inside
  a longer name such as `$Bank`.
- **Recursion into structure.** Substitution reaches into lists and
  mappings, so `range: [$LO, $HI]` and nested parameter maps expand too.
- **Unresolved references fail at load.** A `$name` that the block does not
  define is a load-time error, not a literal string passed through to
  validation — a typo'd variable cannot silently reach the hardware.
- **Numeric values are stringified.** A number in `values` becomes its
  string form before substitution. For unit-bearing parameters — a field, a
  time — give the values as quoted strings, `values: ['3318 G', '3376 G']`,
  so that `value: $B` substitutes a valid `"<value> <unit>"` string. Plain
  numeric parameters (a count, an AWG percent, a rate) accept the substituted
  string directly — the `Int` / `Float` parameters coerce a cleanly-parsing
  string and keep their range checks — so `foreach N in [200, 400]` over
  `points: $N` works as written.

### Loop tagging

Within an iteration the loop tag is stamped into the CSV filenames, so a
field or temperature series produces self-identifying files (for example a
`B_3318G` tag in the name). The manifest additionally records the loop
variable, value, and index on each step run inside the block, so the run
record ties every acquisition back to its position in the series.

### Per-iteration failure policy

`on_fail` on the block governs what happens when a sub-step aborts an
iteration:

- **`continue`** (default) records the failed iteration, notifies, and moves
  on to the next value. A dead field or temperature position must not kill
  the whole series.
- **`abort`** propagates the failure and stops the run, like the global
  abort policy.

Two kinds of abort are never swallowed by `continue`, because repeating them
across every remaining value would only repeat the failure: an explicit
operator decision (a checkpoint abort, an interactive `ask` that chose
abort, or a closed prompt), and an unexpected non-step error (a code bug that
would recur identically each iteration). Both stop the series immediately
regardless of `on_fail`.

The rail-triggered coarse-stage fallback (see
[The tune-up chain](tuning.md)) does not reach inside a `foreach`: the intent
is to tune once before the loop, so a sub-step's amplitude-rail failure is
handled by the block's `on_fail` rather than by re-running an earlier
`tune.power_for_length`.

## Adaptive scan control

The experiment steps (`exp.t2`, `exp.t1`) and the field sweep
(`field.edfs`) can decide their own scan count at run time instead of always
running the full `scans`. In every case `scans` is the ceiling and the
adaptive control can only stop earlier.

**`target_snr`** turns `scans` into a ceiling and stops as soon as the
accumulated data reaches the requested signal-to-noise score. After each
completed scan the runner measures the current curve's SNR with the same
`echo_snr` judge that gates the finished step, and projects — via
√N scaling — how many scans are needed; it stops once the target is met or
the projection stays within the ceiling. The projection aims at 1.15× the
requested SNR, a small margin that guards against `echo_snr` reading
optimistically on the first few scans, so the finished curve reliably clears
the target rather than landing just under it. This is what lets one field
position at the line maximum finish in a few scans while a weak shoulder runs
the full budget:

```yaml
  - exp.t2:
      points: 200
      scans: 48
      target_snr: 10
```

`field.edfs` takes `target_snr` the same way, with `scans` as the ceiling on
the sweep's accumulation.

Because the control only ever *lowers* the scan count — it never adds scans
to chase the target — `target_snr` has no effect when `scans` is left at its
default of 1: there is nothing to shrink. The step announces this as a
warning (visible already in the `--test` pre-flight) so an inert setting is
never carried silently; set `scans` to the largest count you are willing to
spend, and let `target_snr` cut it short.

**`max_duration`** is a wall-clock budget on the experiment steps. If the
projected time for the full `scans` exceeds the budget, the scan count is
reduced mid-run so the run finishes inside it; the data acquired so far is
always kept.

```yaml
  - exp.t2:
      points: 400
      scans: 32
      max_duration: 21600 s
```

When both `target_snr` and `max_duration` are set, the smaller resulting scan count wins. With `adjust_range: true` on T1/T2, SNR stopping and projection wait for the initial range assessment, which takes at most three full scans; the duration and scan ceilings remain active. See [Adaptive relaxation ranges](#adaptive-relaxation-ranges).

## Adaptive relaxation ranges

Set `adjust_range: true` on `exp.t2` or `exp.t1` to assess the measured tail after the first complete scan, before stopping or projecting the scan count from `target_snr`. The default is `false`. A confirmed plateau keeps the same acquisition and all accumulated data, then normal SNR control continues. Excess baseline or a confirmed but short plateau does not repeat the current curve.

If noise makes the first decision uncertain, the runner checks the accumulated curve after up to three full scans, or fewer when the scan ceiling is lower. SNR stopping and projection wait during this initial assessment, while duration and scan limits remain active. If the tail is still uncertain after these scans, accumulation continues with normal SNR control on the current range. The runner reports the final plateau assessment and does not restart a long completed accumulation.

Only a clearly unfinished tail can trigger one early extension of about twice the sampled span. The runner preflights the revised sequence and checks the point ceiling and remaining `max_duration` budget before stopping at the completed scan. If extension is unavailable, the current acquisition continues. Otherwise, the short initial acquisition is saved and one extended acquisition accumulates toward the requested SNR within the remaining time budget. Once the initial acquisition has stopped, the extension runs at least one scan, so `max_duration` can be exceeded by the time needed to save the early data. If the early check itself fails, the step reports `failed` and normal SNR control resumes. Both files are retained; data from their different grids are not combined. The extended acquisition cannot trigger another automatic repair.

The plateau check uses measured late data independently of the fit. Block means must agree with the late reference within 1% of measured early-to-late contrast, including a noise margin; the reference must be stable, and at least one qualifying block must precede it. Fewer than 60 points, invalid data or insufficient contrast cannot support a recommendation. Noise alone does not justify an extension.

After the final curve has a confirmed plateau and passes the hard `relaxation_fit` judge, it recommends a range for the next temperature in the same run: about 55% of actual T2 points on the baseline or 47 T1 points on the recovery plateau. The recommendation contains only range settings. It is kept for the current session and matching sample, experiment kind, field, preset, calibrated pulse settings and seed controls; temperature is excluded. A new run or changed context starts from the protocol's seed. Warming can shorten the next span by at most 25%; cooling or unknown temperature cannot shorten the latest measured span. The current temperature must have passed `temp.wait` before range reuse when temperature state exists. RV movement that invalidates fine pulse calibration clears the recommendation. Dry-runs, failed fits, skipped steps and Stop do not update it. The current integration window and receiver phase are applied afresh.

```yaml
  - foreach:
      var: T
      values: [80, 100, 120, 140]
      steps:
        - temp.set:
            setpoint: $T
        - temp.wait:
            band: 0.3
        - tune.echo_window
        - tune.auto_phase
        - exp.t2:
            tau_start: 300 ns
            tau_step: 20 ns
            points: 400
            scans: 64
            target_snr: 20
            max_duration: 600 s
            adjust_range: true
```

Here `scans: 64` is the ceiling for each acquisition, not a mandatory count. A suitable range keeps its initial scans while accumulating toward SNR 20. A clearly insufficient range is replaced after the early assessment, before spending the full SNR budget. Scan/time limits and an optimistic SNR projection can leave the final SNR below the target. The shipped [T1/T2](examples.md#a-temperature-series-t1-and-t2) and [T2-only](examples.md#a-temperature-series-t2-only) examples include a starting-temperature fine calibration. Edit their sample, field, temperature values and preset choices for the actual setup.

T2 keeps its selected repetition rate; the T2-only example refreshes `tune.rep_rate` at each temperature before using `rep_rate: auto`. The first seeded T1 acquisition uses the protocol's explicit, automatic or preset rate. A carried or repaired T1 range recalculates the maximum timing-compatible rate in 0.1 Hz steps using the full Log Time worker/driver preflight across all points; for Nd:YAG this resolves to the fixed 9.9 Hz rate and fails if the sequence does not fit that period. The cache does not carry an old `tune.rep_rate` recommendation. Timing compatibility does not prove physical recovery between shots.

`adjust_max_points` caps automatically resized sweeps at 4096 requested points by default (allowed range: 60–100000); it does not trim an unchanged range or the initial protocol range. T2 preserves grid spacing and T1 logarithmic density where possible; the actual T1 grid may contain fewer points after rounding and deduplication. The `range_adjustment` result records the early check, measured coverage, reason and both CSV paths if there was an extension. Top-level `start_s` and `end_s` are saved-axis bounds; T1's `t_start` and `t_end` control the log grid. The final curve supplies the fit and hard judge. `max_duration` covers analysis, preflights and both acquisitions as a shared projected budget, not a hard deadline; Stop aborts normally. In `--test` mode canned data cannot establish measured range carryover.

## rep_rate: auto

`exp.t1`, `exp.t2`, `tune.find_echo` and `tune.maximize_echo` accept `rep_rate` as a number in Hz or the literal `auto`. Without it, the experiment steps and first echo search use the preset rate; maximization inherits the search rate. Preliminary rates, including resolved automatic values, must lie within 0.1–10000 Hz. `rep_rate: auto` uses the recommendation stored by an earlier `tune.rep_rate` step; if no `tune.rep_rate` result is in the session it is an error, so `auto` requires `tune.rep_rate` to have run first. `epr-auto validate` catches the statically dead case — a `rep_rate: auto` with no earlier `tune.rep_rate` in the step order — as a load-time **warning**, so you see it at your desk rather than at the abort. The runner still checks that the sweep fits one repetition period — a T1 sweep, for instance, needs `1/rep_rate` beyond `t_end` plus the sequence tail.

```yaml
  - tune.rep_rate
  - exp.t2:
      points: 400
      rep_rate: auto
```

For a new preliminary setup, find an echo at an explicit or preset rate before measuring its saturation. Use the same echo preset for the scan and maximization:

```yaml
  - tune.find_echo:
      preset: hahn_echo_4s.phase_awg
      center: 3445 G
      span: 100 G
  - tune.rep_rate:
      preset: hahn_echo_4s.phase_awg
      mode: quantitative
      rate_max: 2000
  - tune.maximize_echo:
      preset: hahn_echo_4s.phase_awg
      rep_rate: auto
```

This fragment follows the ringing check and optional resonator scan in the [preliminary workflow](tuning.md#preliminary-tuning). Automatic rate selection consumes a stored recommendation; it does not start an implicit scan. All four exported presets carry the rate used by maximization.

For a focused live-rate check at a fixed field, use [`rep_rate_live.yaml`](https://github.com/Anatoly1010/Atomize_ITC/blob/main/protocols/rep_rate_live.yaml): it sets 3318 G, phases once, then tests 10–2000 Hz with six rates, three returned curves per stable group, one group, a 120 s per-rate timeout and quantitative fitting. The card remains open across the grid; a timeout or Stop preserves the partial live history.

## Retries versus judges

Two independent mechanisms decide whether a step succeeds, and they answer
different questions. `retries` and `on_fail` handle a step that *failed* —
bad data, a rejected fit, an engine or lock error — by re-running it and then
deciding abort/skip/ask. Judges handle whether a step's result is *good
enough*: every tuning, field, and experiment primitive returns judge reports,
and in a live run a failed **hard** judge (echo SNR, fit quality on the
relaxation steps, the amplitude rails, and so on) raises a step failure that
feeds straight into the retry/`on_fail` machinery, while **advisory** judges
(a compression-linearity diagnostic, the global nutation-fit quality, the
coarse-stage convergence diagnostic) only warn and never abort. In a dry-run
all judges are logged but none abort, so `--test` shows you the diagnostics
without stopping the run.

GUI dry runs retain operator dialogs, so Continue, Skip, Abort and Stop can be tested with canned data. See [Running from the main window](quickstart.md#running-from-the-main-window).
