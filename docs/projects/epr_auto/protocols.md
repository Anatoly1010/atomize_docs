# Writing protocols

A protocol is a single YAML file that lists the steps `epr_auto` runs, in
order, together with the autonomy level and a few run-wide settings. This
page is the reference for that YAML dialect: the top-level keys, how a step
is written, the value syntax every parameter shares, the per-step failure
and checkpoint controls, and the `foreach` series block. For the parameters
of each individual step see the [Step reference](steps.md); for the tuning
chain the steps compose into, see [The tune-up chain](tuning.md).

Validate a file at any time without running it:

```bash
epr-auto validate my_protocol.yaml
```

Validation is thorough: it resolves every preset path, range-checks every
parameter, expands `foreach` blocks, and reports the first problem as
`INVALID: <file>:<line>: <message>`. A protocol that validates will load; it
does not guarantee the hardware will cooperate.

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — everything on this page is
    implemented and verified in dry-run (`--test`) mode. Always dry-run a
    protocol first, and keep the first live sessions in `supervised`
    autonomy with an operator present.

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
  - tune.auto_phase
  - field.edfs:
      range: [338 mT, 352 mT]
      pick: max
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

**`auto`** is a literal keyword accepted by a few parameters in place of an
explicit value: `range: auto` on `field.edfs` centres the sweep on the
resonance computed from the synthesizer readout, and `rep_rate: auto` on the
experiment steps pulls in the `tune.rep_rate` recommendation (see below).

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
- **`ask`** prompts the operator at the terminal to retry, skip, or abort.
  This needs an attached terminal and an attended run: in a dry-run, in
  `autonomous` mode, or with no tty, `ask` degrades to `abort` (and notifies
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

A checkpoint that would pause but has no terminal to prompt at — an
unattended run in `supervised` or `checkpointed` mode — is a hard abort, not
a silent continue, so a batch job cannot slip past a confirmation the author
demanded. In a dry-run every checkpoint and every operator prompt is
auto-continued and logged, so `--test` exercises the full step list without
stopping.

## Series: the foreach block

A `foreach` block runs its sub-steps once for each value of a loop variable,
substituting the value into the sub-steps. It is how a field series or a
temperature series is written: tune once, then repeat a measurement group
across a list of positions.

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
  string form before substitution. Because unit-bearing parameters require a
  `"<value> <unit>"` string, give those values as quoted strings —
  `values: ['3318 G', '3376 G']`, not bare numbers — so that `value: $B`
  substitutes a valid field string.

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
the projection stays within the ceiling. This is what lets one field
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

When both `target_snr` and `max_duration` are set, the smaller resulting scan
count wins — the run stops at whichever limit it reaches first.

## rep_rate: auto

The experiment steps accept `rep_rate` as a number in Hz (defaulting to the
preset's own value) or the literal `auto`. `rep_rate: auto` uses the
recommendation stored by an earlier `tune.rep_rate` step; if no
`tune.rep_rate` result is in the session it is an error, so `auto` requires
`tune.rep_rate` to have run first. The runner still checks that the sweep fits
one repetition period — a T1 sweep, for instance, needs `1/rep_rate` beyond
`t_end` plus the sequence tail.

```yaml
  - tune.rep_rate
  - exp.t2:
      points: 400
      rep_rate: auto
```

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
