# Step reference

<!-- AUTO-GENERATED — do not edit by hand.
     Regenerate from the Atomize_ITC repo:
     python3 -m atomize.epr_auto.docgen docs/projects/epr_auto/steps.md
     (run from atomize_docs' parent layout; pass the real output path) -->

Every step a protocol can name, with its parameters, defaults and
constraints — generated from the runner's own step registry, so this page
cannot drift from the code. The same listing is available offline from
`epr-auto steps`. How the steps compose into a protocol is described in
[Writing protocols](protocols.md).

A parameter marked *required* has no default and must appear in the
protocol; every other parameter may be omitted. Time and field values are
the framework-wide `"<value> <unit>"` strings (`ns/us/ms/s`, `G/mT/T`).

## Tuning steps

### tune.auto_phase

Acquire an echo and zero the signal phase (principal-axis auto_phase_zero).

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | echo preset the phase is measured on |
| `points` | integer (>= 2) | `4` | sweep points for the quick phase acquisition |
| `scans` | integer (>= 1) | `1` | scans for the quick acquisition |

### tune.echo_window

Set the integration window from an averaged echo trace (center = smoothed |V| max, width = FWHM x factor); run BEFORE tune.auto_phase.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | echo preset the trace is taken with |
| `factor` | number (1..10) | `2.0` | window width as a multiple of the echo FWHM |
| `sweeps` | integer (>= 1) | `3` | full phase cycles to average for the trace |

### tune.pi_calibration

Fine stage: sweep AWG amplitude at fixed length (default) or length nutation; fit pi and pi/2 independently.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | — | defaults to ampl_4s / rabi_echo_4s by mode |
| `mode` | amplitude \| length | `amplitude` | amplitude: sweep AWG amplitude (%) at fixed length; length: linear-time nutation (ns) |
| `channel` | AWG | `AWG` | AWG only (RECT is a later phase) |
| `points` | integer (>= 2) | — | sweep points (default: preset value) |
| `scans` | integer (>= 1) | — | scans (default: preset value) |
| `step` | number (>= 0.01) | — | amplitude step in % (Amplitude mode; default: preset value) |
| `refine` | boolean | `False` | re-run the nutation once with the re-scaled soft detection pair (new-sample insurance) |

### tune.power_for_length

Coarse stage: step the rotary vane until pi lands at the target length (see ARCHITECTURE.md "Flip-angle knobs").

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `target_length` | time ("300 ns") | *required* | desired pi pulse length |
| `amplitude` | number (1..100) | `95.0` | AWG amplitude (%) held during the vane scan |
| `preset` | preset file | `rabi_echo_4s.phase_awg` | length-nutation preset used to measure pi |
| `tolerance` | time ("300 ns") | `3.2 ns` | accept pi within this of target_length |
| `max_iter` | integer (>= 1) | `4` | vane iterations before giving up (the reported state is the one the vane is actually in) |
| `rehome` | no \| limit | `no` | limit: true re-home at the 60 dB switch first |

### tune.rep_rate

Repetition-rate saturation scan: quick echo per rate on a log grid, fit A = A0*(1 - exp(-T/T1_eff)); stores the recommendation for the exp.* steps' rep_rate: auto.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | echo preset for the per-rate quick acquisitions |
| `rate_min` | number (0.1..100000) | `20.0` | slowest rate (Hz) — must reach the unsaturated plateau |
| `rate_max` | number (0.1..100000) | `2000.0` | fastest rate (Hz); recommendations are never extrapolated above it |
| `steps` | integer (3..20) | `6` | log-grid rates between rate_min and rate_max |
| `points` | integer (>= 2) | `4` | sweep points per quick acquisition |
| `scans` | integer (>= 1) | `1` | scans per quick acquisition |
| `factor` | number (1..20) | `5.0` | quantitative-mode period = factor x T1_eff (5 -> <1% residual saturation) |
| `mode` | quantitative \| sensitivity | `quantitative` | sensitivity: period = 1.26 x T1_eff, max S/sqrt(time) — tuning/EDFS only, NOT for quantitative relaxation runs |

## Field steps

### field.edfs

Echo-detected field sweep; pick the working field and set the magnet. range: auto centers on h·ν/(g·μ_B) at ν = ν_LO − ν_IF (the LO readout minus the preset's AWG intermediate frequency).

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `ed_4s.phase_awg` | field-sweep echo-detection preset |
| `range` | 'auto' \| [field ("3478 G"), field ("3478 G")] | *required* | [start, end] field, or 'auto' |
| `points` | integer (>= 2) | `200` | field points across the sweep range |
| `scans` | integer (>= 1) | `1` | scan count (the ceiling when target_snr stops the sweep early) |
| `pick` | max \| marker \| value | `max` | working field: max = magnitude maximum of the sweep; value = the 'value' parameter (marker needs the interactive tools) |
| `value` | field ("3478 G") | — | field to set when pick: value |
| `g` | number (0.1..20) | `2.0023` | g-factor for the range: auto center |
| `span` | field ("3478 G") | `250 G` | half-width of the range: auto sweep |
| `offset` | field offset ("-15 G") | `0 G` | known magnet-calibration shift added to the range: auto center |
| `target_snr` | number (>= 3) | — | SNR-driven scan count: scans becomes the ceiling; stop early once the accumulated sweep reaches this echo_snr score (min = the judge pass floor: a lower target would stop on a sweep the hard judge then rejects) |

### field.set

Set the magnetic field directly.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `value` | field ("3478 G") | *required* | field to set, e.g. "3318 G" |

## Temperature steps

### temp.set

Set the Lakeshore 335 setpoint (and heater range); returns immediately — pair with temp.wait.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `setpoint` | number (0.1..400) | *required* | kelvin |
| `heater_range` | Off \| 0.5 W \| 5 W \| 50 W | — | unchanged when omitted |
| `rephase_delta` | number (>= 0) | `1.0` | invalidate auto_phase and the tune.rep_rate recommendation once the setpoint moves this many K from where each was measured (0 = any change; the oTP series showed ~1 deg of zero-order swing per K, and T1 itself is strongly temperature-dependent) |

### temp.wait

Wait until the temperature holds inside the band (temp_control setter-waiter semantics); timeout fails the step.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `band` | number (>= 0.01) | `0.2` | +/- kelvin around the setpoint |
| `channels` | A \| B \| AB | `B` | thermometer channel(s) that must hold inside the band |
| `hold` | integer (>= 1) | `3` | consecutive in-band polls (1 s cadence) required |
| `timeout` | time ("300 ns") | `1800 s` | wall-clock limit; exceeding it fails the step |
| `setpoint` | number (0.1..400) | — | default: the setpoint already on the device |
| `rephase_delta` | number (>= 0) | `1.0` | invalidate auto_phase and the tune.rep_rate recommendation once the setpoint moves this many K from where each was measured (0 = any change; the oTP series showed ~1 deg of zero-order swing per K, and T1 itself is strongly temperature-dependent) |

## Experiment steps

### exp.t1

Inversion recovery (T1), log-time sweep, with fit.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | <code>inversion_recovery_echo_4s_log<wbr>.phase_awg</code> | Log Time inversion-recovery preset |
| `t_start` | time ("300 ns") | `500 ns` | shortest recovery delay; also sets the log-spacing density |
| `t_end` | time ("300 ns") | `5 ms` | longest recovery delay — physically several times the expected T1 |
| `points` | integer (>= 2) | *required* | log-grid points; the worker deduplicates the grid-rounded axis, so the saved curve may hold fewer |
| `scans` | integer (>= 1) | `1` | scan count — the ceiling when target_snr or max_duration shrink the run |
| `window` | auto \| preset | `auto` | auto: tune.echo_window result; preset: stored values |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = inferred from the preset amplitude levels |
| `max_duration` | time ("300 ns") | — | wall-clock budget; the scan count shrinks mid-run to finish inside it (data acquired so far is kept) |
| `rep_rate` | 'auto' \| number | — | repetition rate in Hz (default: preset value); 'auto' = the tune.rep_rate recommendation; a T1 sweep needs 1/rep_rate beyond t_end plus the sequence tail |
| `target_snr` | number (>= 1) | — | SNR-driven scan count: scans becomes the ceiling; stop early once the accumulated curve reaches this echo_snr score (min wins vs max_duration) |

### exp.t2

Hahn echo decay (T2/Tm), linear tau sweep, with fit.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | Linear Time moving-echo (Hahn) preset |
| `tau_start` | time ("300 ns") | `300 ns` | first tau; the saved axis is the evolution time 2*tau |
| `tau_step` | time ("300 ns") | `12 ns` | tau increment per point |
| `points` | integer (>= 2) | *required* | sweep points |
| `scans` | integer (>= 1) | `1` | scan count — the ceiling when target_snr or max_duration shrink the run |
| `window` | auto \| preset | `auto` | auto: tune.echo_window result; preset: stored values |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = inferred from the preset amplitude levels |
| `max_duration` | time ("300 ns") | — | wall-clock budget; the scan count shrinks mid-run to finish inside it (data acquired so far is kept) |
| `rep_rate` | 'auto' \| number | — | repetition rate in Hz (default: preset value); 'auto' = the tune.rep_rate recommendation; the sweep must fit one period |
| `target_snr` | number (>= 1) | — | SNR-driven scan count: scans becomes the ceiling; stop early once the accumulated curve reaches this echo_snr score (min wins vs max_duration) |

