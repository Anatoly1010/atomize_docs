# Quickstart

This page takes you from a fresh checkout to a dry-run of a protocol, then
describes the prerequisites of a live run on the real spectrometer. It
assumes you have already read the
[overview](index.md); the individual steps a protocol can call are catalogued
in the [step reference](steps.md).

## Install

`epr_auto` ships **only** with Atomize_ITC — the EPR-endstation variant of
Atomize. It is not part of plain Atomize and is not published to PyPI, so the
only way to get it is a source checkout of the ITC repository. Clone it and
install in editable mode, exactly as for the rest of the endstation build:

```bash
git clone https://github.com/Anatoly1010/Atomize_ITC.git
cd Atomize_ITC
pip3 install -e .
```

The editable install registers the console entry point `epr-auto`
(defined in `pyproject.toml` as `atomize.epr_auto.cli:main`). Everything below
can be typed either way — the two forms are identical:

```bash
epr-auto <command> ...
python3 -m atomize.epr_auto <command> ...
```

!!! note
    The machine at the endstation has `python3` only — there is no `python`
    on `PATH`. Use `python3` and `pip3` throughout.

## The three commands

The runner exposes three subcommands.

| Command | Purpose |
| ------- | ------- |
| `epr-auto steps` | list every step and its parameters |
| `epr-auto validate <file>.yaml` | parse and check a protocol without running it |
| `epr-auto run <file>.yaml [--test]` | execute a protocol (`--test` = dry-run) |

### `epr-auto steps`

Prints the step registry — the same catalogue rendered on the
[step reference](steps.md) page, but generated live from the code, so it can
never drift. Each entry gives the step name, a one-line summary, and every
parameter with its type, default (in brackets) and help text:

```text
exp.t2
    Hahn echo decay (T2/Tm), linear tau sweep, with fit
    preset: preset file [hahn_echo_4s.phase_awg] — Linear Time moving-echo (Hahn) preset (saved with 'IQ Correction: 2')
    tau_start: time ("300 ns") [300 ns] — first tau; the saved axis is the evolution time 2*tau
    tau_step: time ("300 ns") [12 ns] — tau increment per point
    points: integer, required — sweep points
    scans: integer [1] — scan count — the ceiling when target_snr or max_duration shrink the run
    ...
```

### `epr-auto validate`

Loads the protocol, resolves every preset path, substitutes the `foreach`
loop variables and range-checks every parameter — all the load-time checks a
`run` performs — but stops before executing anything. Every error is reported
as `<file>:<line>: message`, so a typo in a parameter or a preset that
resolves nowhere is caught at your desk rather than at the bench. On success it
echoes a one-line summary of what it parsed:

```text
OK: field_series_t1t2.yaml — sample 'field_series', autonomy checkpointed, 5 entries (field.edfs, tune.echo_window, tune.auto_phase, tune.pi_calibration, foreach[B])
```

### `epr-auto run --test`

Runs the protocol end-to-end in dry-run mode. This is the pre-flight check you
run before every bench session:

```bash
epr-auto run protocols/overnight_t2.yaml --test
```

```text
=== overnight_t2.yaml | sample: test_sample | autonomy: checkpointed | 4 steps | DRY-RUN (test mode) ===
[1/4] tune.auto_phase  (line 8)
      preset = /home/.../experiments/hahn_echo_4s.phase_awg
      points = 4
      scans = 1
      [PASS] phase_coherence: score=inf (note=dry-run, not judged)
      -> phase_deg=0.0, zero_order_deg=0.0, temperature_k=200.0, canned=True
[2/4] field.edfs  (line 10)
      ...
      [checkpoint] would pause here (checkpointed mode) — auto-continuing in dry-run
      -> field=3450.0 G, pick=max, canned=True
...
=== finished: 4/4 steps ran ===
```

## What a dry-run actually does

`--test` is not a syntax check — it exercises the whole framework's **test
mode**, the same pre-flight path used throughout Atomize. Before importing any
device module, the CLI rewrites `sys.argv[1]` to `'test'`; every device module
reads that flag in its constructor and takes its test branch, so no hardware is
touched and no main-window GUI is needed. Concretely, a dry-run:

- instantiates the pulser, bridge, field and temperature controllers as
  **test-mode devices** that return canned readings instead of talking to the
  instruments (each step's result is flagged `canned=True` in the log);
- **pre-flights every step**: the argument range-checks and the sequence
  overlap/length asserts that only fire in test mode run for real, so an
  illegal pulse geometry or an out-of-range setting is rejected here rather
  than at the bench;
- **computes and logs every judge but does not gate on it** — the log shows
  `[PASS] ... (note=dry-run, not judged)`, so a dry-run always runs to the end
  regardless of the (canned) scores;
- flows canned results **between** steps through the session state, so a
  calibration produced by an earlier step is consumed by a later one exactly as
  it would be live;
- **auto-continues at every checkpoint** (`[checkpoint] would pause here …`)
  instead of prompting, and writes **no** `manifest.json` — the run record is
  log-only in test mode so a dry-run never litters the data directory.

The result is that a protocol is walked from first step to last, with all its
validation and cross-step plumbing exercised, on a machine with no spectrometer
attached.

## Running live

!!! warning "Commissioning status"
    Live execution is enabled from the CLI, but the automation chain has not
    yet been validated on the spectrometer — every step is implemented and
    verified in dry-run (`--test`) only. Always dry-run a protocol first, and
    keep the first live sessions in `supervised` autonomy with an operator
    present.

Prepare a live session as follows:

- **Launch the main Atomize GUI first, and keep it open.** A live run's
  acquisition worker pushes its traces to the main window's LivePlot server; a
  real run dies without it. (A dry-run needs no GUI — see above.)
- **Close the interactive field and temperature tools.** At run start the
  runner seizes the `field.param` and `temp_param` cross-process locks as
  `epr_auto`, the same discipline the four experiment-runner GUIs use to keep
  the interactive tools off the GPIB devices. If either lock is already held by
  another tool, the runner refuses to start; it releases the locks in `finally`
  and on `atexit`, so a crash never leaves them stranded.
- **Run from the repository root.** The CLI changes into `libs/` itself,
  because the Insys FPGA driver reads `brd.ini` / `exam_adc.ini` relative to
  the working directory at instantiation time; it also re-provisions the device
  config store and pre-flights every step in test mode before touching
  hardware.
- **Start in supervised autonomy until the chain is trusted.** In
  `autonomy: supervised` the runner pauses before every step (press Enter to
  continue), so you can watch each move; move up to `checkpointed` and then
  `autonomous` once you trust the protocol.

Always dry-run a protocol with `--test` before its first live run.

## The run directory

A live run writes everything for that session into one directory. By default it
is:

```text
~/epr_data/epr_auto_<date>_<sample>/
```

where `<date>` is today's date (`YYYY-MM-DD`) and `<sample>` is the protocol's
`sample:` field with unsafe characters replaced by underscores. A protocol can
override the location with an `output:` template in which `{date}` and
`{sample}` expand (for example `output: ~/epr_data/{date}_{sample}`).

The directory holds three kinds of file:

- **`manifest.json`** — the crash-safe run record, rewritten atomically after
  every step so an interrupted run still leaves a valid file. It carries the
  protocol name, sample, autonomy mode, start/finish timestamps and overall
  status, plus one entry per executed step: its resolved parameters, result,
  judge reports, attempt count, and — inside a `foreach` — the loop
  `{var, value, index}`.
- **A copy of the protocol**, saved as `protocol_<name>.yaml`, so the exact
  YAML that produced the data always sits next to it.
- **The acquisition CSVs**, named `NNN_tag.csv` with a per-session counter, for
  example `001_t2.csv`. Inside a `foreach` iteration the loop stamp is folded
  into the name, so a field or temperature series is self-identifying —
  `003_t2_B_3318G.csv` is the third saved file, a T2 measurement, at the
  `B = 3318 G` point of the loop.

!!! note
    `manifest.json` and the CSVs are written only on a **live** run. A dry-run
    logs the same information to the terminal but writes nothing to disk.

## Next steps

- [Writing protocols](protocols.md) — the YAML schema in full: `autonomy`,
  `on_fail`/`retries`, `checkpoint`, the `foreach` series block and value
  substitution.
- [The tune-up chain](tuning.md) — how the `tune.*` steps compose into a
  calibration sequence and when each calibration is invalidated.
