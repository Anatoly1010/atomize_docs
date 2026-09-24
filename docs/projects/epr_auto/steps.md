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

## MW bridge steps

### bridge.set

Set RV, synthesizer and/or video attenuation with settling.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `attenuation_db` | number (0..60) | — | rotary-vane (RV) attenuation of microwave excitation, in dB |
| `frequency_mhz` | integer (7000..12000) | — |  |
| `video1_db` | number (0..30) | — | receiver Video Attenuation 1 (VA1), in 2 dB increments |
| `video2_db` | number (0..31.5) | — | receiver Video Attenuation 2 (VA2), in 0.5 dB increments |

## Tuning steps

### tune.apply_calibration

Write the session calibration, zero-order phase, echo window and field into a preset file.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | *required* | preset file to rewrite in place, or to copy from when destination is given |
| `pulse_map` | mapping {P2..P9: pi \| pi2} \| 'none' | — | pi2/pi roles; inferred from the preset when omitted |
| `destination` | string | — | absolute path of the file to write instead of rewriting the preset in place |

### tune.auto_phase

Acquire an echo and zero the signal phase (principal-axis auto_phase_zero).

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | echo preset the phase is measured on |
| `points` | integer (>= 2) | `16` | sweep points for the quick phase acquisition (the phase_coherence judge needs >= ~10 to be informative — its noise floor is 3/sqrt(n)) |
| `scans` | integer (>= 1) | `1` | scans for the quick acquisition |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = patch from the session pi_calibration when one exists (inferred from the preset amplitude levels), else the stored values |

### tune.echo_window

Set the integration window from an averaged echo trace (center = smoothed |V| max, width = FWHM x factor); run BEFORE tune.auto_phase.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | echo preset the trace is taken with |
| `factor` | number (1..10) | `2.0` | window width as a multiple of the echo FWHM |
| `sweeps` | integer (>= 1) | `3` | minimum full phase cycles to average for the trace |
| `search_from` | time ("300 ns") | `200 ns` | start the echo search at this time relative to DETECTION; exclude early receiver transients while retaining the echo |
| `min_width` | time ("300 ns") | `20 ns` | reject a peak whose FWHM is below this as a transient (masked out, the search goes on); nothing wider left = the echo_in_trace judge fails |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = patch from the session pi_calibration when one exists (inferred from the preset amplitude levels), else the stored values |

### tune.find_echo

Full-window magnitude field search, then resolve the echo window.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | AWG SINE echo preset; defines the IF |
| `center` | field ("3478 G") | *required* |  |
| `span` | field ("3478 G") | *required* |  |
| `points` | integer (7..1001) | `41` |  |
| `attenuation_db` | number (0..60) | `10` | fixed RV for the echo search and maximization |
| `frequency_shift_mhz` | integer | `0` | signed shift from resonator center, or current bridge frequency without a scan |
| `pulse_length` | time ("300 ns") | — | target pi pulse length; every echo pulse takes it (default: the preset's shortest MW pulse) |
| `adjust_video` | boolean | `True` | adjust video attenuation to keep the echo at or below 200 mV |
| `rep_rate` | 'auto' \| number | — | repetition rate in Hz, 0.1–10000; 'auto' uses an earlier tune.rep_rate recommendation within that range; omitted keeps the preset value |
| `scans` | integer (1..100) | `1` |  |
| `averages` | integer (1..10000) | `10` |  |
| `search_from` | time ("300 ns") | `200 ns` |  |
| `min_width` | time ("300 ns") | `20 ns` |  |

### tune.maximize_echo

Fixed-RV amplitude scan (pi/2 at a, pi at 2a), then field refinement.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | AWG SINE echo preset; defines the IF |
| `attenuation_db` | number (0..60) | — | fixed RV; defaults to the find_echo setting |
| `pulse_length` | time ("300 ns") | — | target pi pulse length for all echo pulses; defaults to the find_echo setting |
| `amplitude_range` | [number, number] | `[5, 50]` | pi/2 amplitude bounds in % |
| `coarse_step` | number (1..25) | `5` |  |
| `fine_step` | number (0.5..5) | `1` |  |
| `field_span` | field ("3478 G") | `10 G` |  |
| `points` | integer (7..1001) | `21` |  |
| `improvement` | number (0.001..1) | `0.05` |  |
| `pulse_map` | mapping {P2..P9: pi \| pi2} \| 'none' | — | pi2/pi roles, e.g. {P2: pi2, P3: pi}; inferred from the preset when omitted |
| `adjust_video` | boolean | — | omit to inherit tune.find_echo; true adjusts video attenuation to 200 mV |
| `rep_rate` | 'auto' \| number | — | repetition rate in Hz, 0.1–10000; 'auto' uses an earlier tune.rep_rate recommendation within that range; omitted inherits tune.find_echo |
| `scans` | integer (1..100) | `1` |  |
| `averages` | integer (1..10000) | `10` |  |
| `search_from` | time ("300 ns") | `200 ns` |  |
| `min_width` | time ("300 ns") | `20 ns` |  |

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

Fixed-tau live repetition-rate scan with a temporary 512 KB ADC buffer: consecutive fresh curves stable to 5%, fit A = A0*(1 - exp(-T/T1_eff)); stores the recommendation for preliminary tuning and exp.* steps using rep_rate: auto.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | two-pulse echo preset; tau remains fixed during live tuning |
| `rate_min` | number (10..100000) | `10.0` | slowest rate (Hz), at least 10 — must reach the unsaturated plateau |
| `rate_max` | number (10..100000) | `2000.0` | fastest rate (Hz); recommendations are never extrapolated above it |
| `steps` | integer (3..20) | `6` | log-grid rates between rate_min and rate_max |
| `points` | integer (>= 3) | `3` | consecutive fresh live curves within 5%; no tau sweep |
| `scans` | integer (>= 1) | `1` | disjoint stable groups required per rate; all groups must agree within 5% |
| `max_wait` | time ("300 ns") | `120 s` | time limit per rate, including arrival of fresh ADC buffers |
| `factor` | number (1..20) | `5.0` | quantitative-mode period = factor x T1_eff (5 -> <1% residual saturation) |
| `mode` | quantitative \| sensitivity | `quantitative` | sensitivity: period = 1.26 x T1_eff, max S/sqrt(time) — tuning/EDFS only, NOT for quantitative relaxation runs |

### tune.resonator

AWG SINE diode scan; choose a stable early-ringing frequency maximum.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `if_mhz` | integer (1..280) | `50` | built-in SINE IF; must match the later echo preset DETECTION IF |
| `start_mhz` | integer (7000..12000) | `9200` |  |
| `end_mhz` | integer (7000..12000) | `9600` |  |
| `step_mhz` | integer (>= 1) | `1` |  |
| `pulse_length` | time ("300 ns") | `102.4 ns` |  |
| `window` | time ("300 ns") | `2 ns` |  |
| `precision_mhz` | number (>= 1) | `5` |  |
| `min_snr` | number (>= 3) | `5` |  |
| `competitor_ratio` | number (0.1..1) | `0.8` |  |
| `region` | [time ("300 ns"), time ("300 ns")] | — | optional trailing-edge region in trace coordinates |
| `clip_mv` | number (>= 0.001) | — | scope voltage clipping level, when known |
| `scans` | integer (1..100) | `1` |  |
| `averages` | integer (1..10000) | `10` |  |

### tune.ringing_check

Home RV; check magnitude at each of 60,40,20,10,5,0 dB; hard-stop above 100 mV.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `if_mhz` | integer (1..280) | `50` | built-in SINE IF; must match the later echo preset DETECTION IF |
| `pulse_length` | time ("300 ns") | `102.4 ns` | SINE pulse length of the ladder |
| `field` | field ("3478 G") | `100 G` | nonresonant field for the ringing ladder |
| `done` | boolean | `False` | the ladder already passed at this IF; record the limits, move nothing |

### tune.save_presets

Export echo/calibration/field preset copies and a fine-tuning YAML handoff.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | `hahn_echo_4s.phase_awg` | AWG SINE echo preset; defines the IF |
| `calibration_preset` | preset file | `ampl_4s.phase_awg` |  |
| `field_preset` | preset file | `ed_4s.phase_awg` |  |
| `field_span` | field ("3478 G") | — | EDFS span of the handoff, centered on the tuned field; default: the find_echo span |
| `field_points` | integer (2..5001) | `200` |  |
| `calibration_length` | time ("300 ns") | — | target length of the Rabi pulse the fine calibration sweeps; default: the preliminary pulse length |
| `publish_dir` | directory path (relative to the protocol file) | `tuned` | where the handoff (fine_tuning.yaml and its presets) is published; the run directory keeps an archive copy |

### tune.video_attenuation

Adjust video attenuation on the final preset, preserving pulse lengths and zeroing sweep increments.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | *required* |  |
| `adjust_video` | boolean | — | omit to inherit tune.find_echo, or enable adjustment when run standalone |
| `limit_mv` | number (0.001..200) | `200` | maximum allowed echo magnitude in mV |

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
| `offset` | field offset ("-15 G") | `-7.5 G` | known magnet-calibration shift added to the range: auto center; set for your magnet calibration |
| `target_snr` | number (>= 3) | — | SNR-driven scan count: scans becomes the ceiling; stop early once the accumulated sweep reaches this echo_snr score (min = the judge pass floor: a lower target would stop on a sweep the hard judge then rejects) |
| `save_2d` | boolean | `False` | also save the full I/Q matrices of every sweep point as a _2d.h5 file beside the CSV (datasets I, Q, t, sweep); size is points x window samples x 8 bytes |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = patch from the session pi_calibration when one exists (inferred from the preset amplitude levels), else the stored values |

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
| `rephase_delta` | number (>= 0) | `1.0` | invalidate auto_phase and the tune.rep_rate recommendation once the setpoint moves this many K from where each was measured (0 = any change; a measured temperature series showed ~1 deg of zero-order swing per K, and T1 itself is strongly temperature-dependent) |

### temp.wait

Wait until the temperature holds inside the band (temp_control setter-waiter semantics); timeout fails the step.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `band` | number (>= 0.01) | `0.2` | +/- kelvin around the setpoint |
| `channels` | A \| B \| AB | `B` | thermometer channel(s) that must hold inside the band |
| `hold` | integer (>= 1) | `3` | consecutive in-band polls (1 s cadence) required |
| `timeout` | time ("300 ns") | `1800 s` | wall-clock limit; exceeding it fails the step |
| `setpoint` | number (0.1..400) | — | default: the setpoint already on the device |
| `rephase_delta` | number (>= 0) | `1.0` | invalidate auto_phase and the tune.rep_rate recommendation once the setpoint moves this many K from where each was measured (0 = any change; a measured temperature series showed ~1 deg of zero-order swing per K, and T1 itself is strongly temperature-dependent) |

## Experiment steps

### exp.t1

Inversion recovery (T1), log-time sweep, with fit.

| Parameter | Type | Default | Description |
| --------- | ---- | ------- | ----------- |
| `preset` | preset file | <code>inversion_recovery_echo_4s_log<wbr>.phase_awg</code> | Log Time inversion-recovery preset |
| `t_start` | time ("300 ns") | `500 ns` | shortest recovery delay; also sets the log-spacing density |
| `t_end` | time ("300 ns") | `5 ms` | longest recovery delay — physically several times the expected T1 |
| `adjust_range` | boolean | `False` | check the range during the first 1–3 scans before SNR stopping; only a clearly unfinished tail gets one early extension; plan 45–50 plateau points for the next temperature; reused/repaired ranges use the maximum timing-compatible rate |
| `adjust_max_points` | integer (60..100000) | `4096` | point ceiling when automatically resizing a sweep; reduce log-grid density if needed |
| `save_2d` | boolean | `False` | also save the full I/Q matrices of every sweep point as a _2d.h5 file beside the CSV (datasets I, Q, t, sweep); size is points x window samples x 8 bytes |
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
| `adjust_range` | boolean | `False` | check the range during the first 1–3 scans before SNR stopping; only a clearly unfinished tail gets one early extension; plan 50–60% baseline for the next temperature |
| `adjust_max_points` | integer (60..100000) | `4096` | point ceiling when automatically resizing a sweep; increase the grid step if needed |
| `save_2d` | boolean | `False` | also save the full I/Q matrices of every sweep point as a _2d.h5 file beside the CSV (datasets I, Q, t, sweep); size is points x window samples x 8 bytes |
| `points` | integer (>= 2) | *required* | sweep points |
| `scans` | integer (>= 1) | `1` | scan count — the ceiling when target_snr or max_duration shrink the run |
| `window` | auto \| preset | `auto` | auto: tune.echo_window result; preset: stored values |
| `apply_cal` | mapping {P2..P9: pi \| pi2} \| 'none' | — | slot -> pi/pi2 map; none = do not patch; omitted = inferred from the preset amplitude levels |
| `max_duration` | time ("300 ns") | — | wall-clock budget; the scan count shrinks mid-run to finish inside it (data acquired so far is kept) |
| `rep_rate` | 'auto' \| number | — | repetition rate in Hz (default: preset value); 'auto' = the tune.rep_rate recommendation; the sweep must fit one period |
| `target_snr` | number (>= 1) | — | SNR-driven scan count: scans becomes the ceiling; stop early once the accumulated curve reaches this echo_snr score (min wins vs max_duration) |

