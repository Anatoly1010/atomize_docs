# Digitizers

## Devices

- Spectrum M4I **4450 X8**; Tested 08/2021
- Spectrum M4I **2211 X8**; Tested 01/2023

The original [library](https://spectrum-instrumentation.com/en/m4i4450-x8) was
written by Spectrum. The library header files (`pyspcm.py`, `spcm_tools.py`)
should be added to the path directly in the module file:

```python
sys.path.append('/path/to/python/header/of/Spectrum/library')
from pyspcm import *
from spcm_tools import *
```

- [Insys FM214x3GDA](https://www.insys.ru/mezzanine/fm214x3gda) as ADC; Tested 03/2025

    The Insys device is available via `ctypes`. The original library can be
    found [here](https://github.com/Anatoly1010/Atomize_ITC/tree/master/libs).

- L card L-502 as ADC; Tested 03/2022

    The L card device is available via `ctypes`. The original library can be
    found [here](https://www.lcard.ru/products/boards/l-502?qt-ltab=6#qt-ltab).
    The path to the installed library should be provided directly in the
    device module file.

## Functions

### digitizer_name() { #digitizer_name }

```python
digitizer_name() -> str
```

Returns the device name.

---

### digitizer_setup() { #digitizer_setup }

```python
digitizer_setup() -> None
```

!!! example
    `digitizer_setup()` writes all the settings into the digitizer.

Writes all the settings modified by other functions to the digitizer. The
function should be called only without arguments. One must initialize the
settings before calling [`digitizer_get_curve()`](#digitizer_get_curve).

The default settings (if no other function was called) are:

| Setting          | M4I 4450 X8 | M4I 2211 X8 |
| ---------------- | ----------- | ----------- |
| Sample clock     | 500 MHz     | 1250 MHz    |
| Number of points | 128         | 256         |
| Input mode       | `HF`        | —           |

Common defaults for both: clock mode `Internal`; reference clock 100 MHz; card
mode `Single`; trigger channel `External`; trigger mode `Positive`; number of
averages 2; trigger delay 0; enabled channels CH0 and CH1; coupling `DC` on
both; impedance `50` Ω on both; horizontal offset 0 % on both; range `500 mV`
on both; posttrigger points 64.

!!! note
    This function is not available for Insys FM214x3GDA — initialization is
    done by [`pulser_open()`](../pulse_programmer.md#pulser_open) instead.

---

### digitizer_get_curve() { #digitizer_get_curve }

```python
digitizer_get_curve() -> numpy.array, numpy.array
digitizer_get_curve() -> numpy.array, numpy.array, numpy.array
```

!!! example
    `digitizer_get_curve()` runs acquisition and returns the data.

Runs acquisition and returns the data obtained. If two channels are enabled by
[`digitizer_channel()`](#digitizer_channel) the output is three numpy arrays
(`xs`, `data_ch0`, `data_ch1`). If one channel is enabled the output is two
numpy arrays (`xs`, `data_ch0`). The `xs` array is returned in seconds; data
arrays are returned in volts. Call without arguments.

!!! note
    For Insys FM214x3GDA, use the
    [`digitizer_get_curve(points, phases, ...)`](#digitizer_get_curve-points) variant.

---

### digitizer_get_curve(integral=True) { #digitizer_get_curve-integral }

```python
digitizer_get_curve(integral: bool = False) -> float
digitizer_get_curve(integral: bool = True)  -> float, float
```

!!! example
    `digitizer_get_curve(integral=True)` runs acquisition and returns the
    integrated data.

Runs acquisition and returns the data integrated over a window in the
oscillogram (the window is set via [`digitizer_window`](#digitizer_window)).
If two channels are enabled, returns two numbers (`integral_ch0`,
`integral_ch1`); with one channel, returns one number (`integral_ch0`). The
integral is returned in volt-seconds. Default is `False`.

!!! note
    For Insys FM214x3GDA, use the
    [`digitizer_get_curve(points, phases, ...)`](#digitizer_get_curve-points) variant.

---

### digitizer_get_curve(points, phases, ...) { #digitizer_get_curve-points }

```python
digitizer_get_curve(
    points: int,
    phases: int,
    integral: bool = False,
    current_scan: int = 1,
    total_scan: int = 1,
) -> numpy.array, numpy.array
```

!!! example
    ```python
    digitizer_get_curve(100, 2, integral=False, current_scan=1, total_scan=1)
    ```
    runs acquisition and returns the phase-cycled data.

Starts data acquisition and [phase cycles](../pulse_programmer.md#pulser_acquisition_cycle)
the data. `points` is the total number of points in the pulse experiment;
`phases` is the total number of phases. Data is phase-cycled according to the
phase list given in the [`DETECTION` pulse](../pulse_programmer.md#pulser_pulse).
The keyword `integral` integrates the data over the configured window.
`current_scan` and `total_scan` indicate the current/total number of
repetitive scans in the experiment to maximize the efficiency of the
Insys FM214x3GDA. `current_scan` should increase during the experiment.

---

### digitizer_close() { #digitizer_close }

```python
digitizer_close() -> None
```

!!! example
    `digitizer_close()` closes the digitizer driver.

Closes the digitizer driver. Call only without arguments. Should always be
called at the end of an experimental script.

!!! note
    Not available for Insys FM214x3GDA — use
    [`pulser_close()`](../pulse_programmer.md#pulser_close) instead.

---

### digitizer_stop() { #digitizer_stop }

```python
digitizer_stop() -> None
```

!!! example
    `digitizer_stop()` stops the digitizer.

Stops the digitizer. Call only without arguments. Should always be called
before redefining digitizer settings with
[`digitizer_setup()`](#digitizer_setup).

!!! note
    Not available for Insys FM214x3GDA or L card L-502 — there is no need to
    stop these ADCs.

---

### digitizer_number_of_points(*points) { #digitizer_number_of_points }

```python
digitizer_number_of_points(points: int) -> None
digitizer_number_of_points() -> int
```

!!! example
    `digitizer_number_of_points(128)` sets the number of points to 128.

Queries or sets the number of points (samples) in the returned oscillogram.

| Constraint      | M4I 4450 X8 | M4I 2211 X8 |
| --------------- | ----------- | ----------- |
| Divisible by    | 16 samples  | 32 samples  |
| Minimum         | 32 samples  | 64 samples  |
| Default         | 128         | 256         |

If no setting matches the argument, the nearest available value is used and a
warning is printed. The difference between number of points and
[posttrigger points](#digitizer_posttrigger) must be less than 8000; otherwise
the nearest valid value is used and a warning is printed.

!!! note
    Not available for Insys FM214x3GDA — the number of points is controlled
    by the trigger pulse length.

---

### digitizer_posttrigger(*post_points) { #digitizer_posttrigger }

```python
digitizer_posttrigger(post_points: int) -> None
digitizer_posttrigger() -> int
```

!!! example
    `digitizer_posttrigger(64)` sets the number of posttrigger points to 64.

Queries or sets the number of posttrigger (horizontal offset) points. Must be
divisible by 16 samples (M4I 4450 X8) or 32 samples (M4I 2211 X8); minimum is
16 / 32 samples respectively. In [`Average`](#digitizer_card_mode) card mode
the maximum is [number of points](#digitizer_number_of_points) minus 16 / 32
samples. Default value is 64. The difference between number of points and
posttrigger points must be less than 8000.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_channel(*channel) { #digitizer_channel }

```python
digitizer_channel(channel: ['CH0', 'CH1']) -> None
digitizer_channel() -> str
```

!!! example
    `digitizer_channel('CH0', 'CH1')` enables CH0 and CH1.

Enables the specified channel or queries enabled channels. Channel must be
one of `['CH0','CH1']`. Default: both channels enabled.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_sample_rate(*s_rate) { #digitizer_sample_rate }

```python
digitizer_sample_rate(sample_rate: float) -> None
digitizer_sample_rate() -> float
```

!!! example
    `digitizer_sample_rate(500)` sets the digitizer sample rate to 500 MHz.

Queries or sets the digitizer sample rate (in MHz).

| Device      | Minimum     | Maximum  | Available values                  |
| ----------- | ----------- | -------- | --------------------------------- |
| M4I 4450 X8 | 1.907 kHz   | 500 MHz  | `[500, 250, 125, …, 0.001907]`    |
| M4I 2211 X8 | 9.536 kHz   | 1250 MHz | `[1250, 625, 312.5, …, 0.009536]` |

If no setting matches the argument, the nearest available value is used and a
warning is printed.

!!! note
    For Insys FM214x3GDA this function only returns the current sample rate —
    see [`digitizer_decimation()`](#digitizer_decimation). Not available for
    L card L-502.

---

### digitizer_clock_mode(*mode) { #digitizer_clock_mode }

```python
digitizer_clock_mode(mode: ['Internal', 'External']) -> None
digitizer_clock_mode() -> str
```

!!! example
    `digitizer_clock_mode('Internal')` sets the Internal clock mode.

Queries or sets the digitizer clock mode. Must be `'Internal'` or `'External'`.

In Internal mode the sampling clock is generated by a programmable high
precision quartz. In External mode the input is fed through a PLL — acting as
a reference clock input, allowing the driver to either use a copy of the
external clock or generate any sampling clock within the allowed range from
the reference clock. The external reference frequency must be set via
[`digitizer_reference_clock()`](#digitizer_reference_clock). Default:
`'Internal'`.

!!! note
    Not available for Insys FM214x3GDA.

---

### digitizer_reference_clock(*ref_clock) { #digitizer_reference_clock }

```python
digitizer_reference_clock(ref_clock: int) -> None
digitizer_reference_clock() -> int
```

!!! example
    `digitizer_reference_clock(100)` sets the digitizer reference clock to
    100 MHz.

Queries or sets the digitizer reference clock (in MHz) for `External` mode of
[`digitizer_clock_mode()`](#digitizer_clock_mode). Minimum 10 MHz, maximum
100 MHz. Default: 100 MHz.

!!! note
    Not available for Insys FM214x3GDA.

---

### digitizer_card_mode(*mode) { #digitizer_card_mode }

```python
digitizer_card_mode(mode: ['Single', 'Average']) -> None
digitizer_card_mode() -> str
```

!!! example
    `digitizer_card_mode('Single')` sets the Single mode of the digitizer.

Queries or sets the digitizer mode. Must be `'Single'` or `'Average'`.

In `Single` mode, data acquisition runs to on-board memory for one trigger
event. In `Average` (Multi) mode the memory is segmented and one segment is
acquired per trigger; data is then transferred to the PC and either
[averaged](#digitizer_get_curve) or
[averaged and integrated](#digitizer_get_curve-integral). The number of
segments is set by
[`digitizer_number_of_averages()`](#digitizer_number_of_averages). Default:
`'Single'`.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_trigger_channel(*ch) { #digitizer_trigger_channel }

```python
digitizer_trigger_channel(channel: ['Software', 'External']) -> None
digitizer_trigger_channel() -> str
```

!!! example
    `digitizer_trigger_channel('Software')` sets the Software trigger.

Queries or sets the digitizer trigger channel. Must be `'Software'` or
`'External'`. `External` corresponds to the `Trg0` channel of the digitizer.
Default: `'External'`.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_trigger_mode(*mode) { #digitizer_trigger_mode }

```python
digitizer_trigger_mode(mode: ['Positive', 'Negative', 'High', 'Low']) -> None
digitizer_trigger_mode() -> str
```

!!! example
    `digitizer_trigger_mode('Positive')` sets the trigger detection for
    positive edges.

Queries or sets the digitizer trigger mode.

| Mode       | Meaning                                              |
| ---------- | ---------------------------------------------------- |
| `Positive` | Trigger on positive edges (crossing 0 from below).   |
| `Negative` | Trigger on negative edges (crossing 0 from above).   |
| `High`     | Trigger on HIGH levels (signal above 0).             |
| `Low`      | Trigger on LOW levels (signal below 0).              |

Default: `'Positive'`.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_number_of_averages(*averages) { #digitizer_number_of_averages }

```python
digitizer_number_of_averages(averages: int) -> None
digitizer_number_of_averages() -> int
```

!!! example
    `digitizer_number_of_averages(2)` sets the number of averages to 2.

Queries or sets the number of averages for
[`Average`](#digitizer_card_mode) card mode. Maximum 10000. For very large
[number of points](#digitizer_number_of_points), the maximum may be limited
by the digitizer memory (1 GS). Default: 2.

!!! note
    Not available for L card L-502.

---

### digitizer_trigger_delay(*delay) { #digitizer_trigger_delay }

```python
digitizer_trigger_delay(delay: str) -> None
digitizer_trigger_delay() -> str
```

!!! example
    `digitizer_trigger_delay('32 ns')` sets the trigger delay to 32 ns.

Queries or sets the digitizer trigger delay. The delay step is 16 samples
(M4I 4450 X8) or 32 samples (M4I 2211 X8). Non-divisible inputs are rounded
and a warning is printed. Default: `'0 ns'`.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_input_mode(*mode) { #digitizer_input_mode }

```python
digitizer_input_mode(mode: ['HF', 'Buffered']) -> None
digitizer_input_mode() -> str
```

!!! example
    `digitizer_input_mode('HF')` sets the HF input mode.

Queries or sets the input mode for the channels. Must be `'HF'` or
`'Buffered'`; applies to both channels.

- **HF**: 50 Ω path, full bandwidth and best dynamic performance.
- **Buffered**: buffered path with all features but limited bandwidth and
  dynamic performance.

Default: `'HF'`.

!!! note
    Not available for M4I 2211 X8, Insys FM214x3GDA, L card L-502.

---

### digitizer_amplitude(*ampl) { #digitizer_amplitude }

```python
digitizer_amplitude(amplitude: int) -> None
digitizer_amplitude() -> str
```

!!! example
    `digitizer_amplitude(500)` sets the range of the digitizer channels to
    ±500 mV.

Queries or sets the input range of the digitizer channels (in mV). Applies to
both channels.

| Device / mode                                                | Allowed ranges (mV)              |
| ------------------------------------------------------------ | -------------------------------- |
| M4I 4450 X8, [Buffered input mode](#digitizer_input_mode)   | `[200, 500, 1000, 2000, 5000, 10000]` |
| M4I 4450 X8, [HF input mode](#digitizer_input_mode)         | `[500, 1000, 2500, 5000]`        |
| M4I 2211 X8                                                  | `[200, 500, 1000, 2500]`         |

If no setting matches, the nearest is used and a warning is printed. Default:
`500 mV`.

!!! note
    Not available for Insys FM214x3GDA (max ≈ 1500 mV) or L card L-502.

---

### digitizer_offset(*offset) { #digitizer_offset }

```python
digitizer_offset(ch0: str, offset0: int) -> None
digitizer_offset(ch0: str, offset0: int, ch1: str, offset1: int) -> None
digitizer_offset() -> str, str
```

!!! example
    ```python
    digitizer_offset('CH0', '1', 'CH1', '50')
    ```
    sets the offset of CH0 to 1 % of the input range and the offset of CH1 to
    50 % of the input range.

Queries or sets the vertical offset of the channels (as a percentage of the
input range). For M4I 4450 X8 the value of the offset (range × argument) is
always **subtracted** from the signal. Step is 1 %.

!!! warning
    No offset can be used for the 1000 mV and 10000 mV ranges in the
    [Buffered input mode](#digitizer_input_mode) (per M4I 4450 X8 docs).

Default: `0` for both channels.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_coupling(*coupling) { #digitizer_coupling }

```python
digitizer_coupling(ch0: str, coupling0: ['AC', 'DC']) -> None
digitizer_coupling(ch0: str, coupling0: str, ch1: str, coupling1: str) -> None
digitizer_coupling() -> str, str
```

!!! example
    ```python
    digitizer_coupling('CH0', 'AC', 'CH1', 'DC')
    ```
    sets the coupling of CH0 to AC and the coupling of CH1 to DC.

Queries or sets the coupling of the digitizer channels. Coupling must be
`'AC'` or `'DC'`. Default: `'DC'` for both channels.

!!! note
    Not available for Insys FM214x3GDA, L card L-502.

---

### digitizer_impedance(*impedance) { #digitizer_impedance }

```python
digitizer_impedance(ch0: str, impedance0: ['1 M', '50']) -> None
digitizer_impedance(ch0: str, impedance0: str, ch1: str, impedance1: str) -> None
digitizer_impedance() -> str, str
```

!!! example
    ```python
    digitizer_impedance('CH0', '50', 'CH1', '1 M')
    ```
    sets CH0 impedance to 50 Ω and CH1 impedance to 1 MΩ.

Queries or sets the channel impedance. Must be `'1 M'` or `'50'`. In
[HF input mode](#digitizer_input_mode) the impedance is fixed at 50 Ω.
Default: `'50'` for both channels.

!!! note
    Not available for M4I 2211 X8, Insys FM214x3GDA, L card L-502 — for these
    digitizers the impedance is fixed at 50 Ω.

---

### digitizer_decimation(*dec) { #digitizer_decimation }

```python
digitizer_decimation(decimation: int) -> None  # 1, 2, or 4
digitizer_decimation() -> int
```

!!! example
    `digitizer_decimation(4)` sets the decimation coefficient to 4.

Queries or sets the decimation coefficient for Insys FM214x3GDA. Available
coefficients are 1, 2, 4 — corresponding to 0.4, 0.8, and 1.6 ns/point.
Can be used instead of
[`digitizer_sample_rate()`](#digitizer_sample_rate).

!!! warning
    Must be called **before**
    [`pulser_open()`](../pulse_programmer.md#pulser_open).

---

### digitizer_flow(*flow) { #digitizer_flow }

```python
digitizer_flow(flow: ['ADC', 'DIN', 'DAC1', 'DAC2', 'DOUT', 'AIN', 'AOUT']) -> None
```

!!! example
    `digitizer_flow()` reads the enabled flow of data.

Queries or sets enabled flows of data. Available options:
`['ADC', 'DIN', 'DAC1', 'DAC2', 'DOUT', 'AIN', 'AOUT']`. Default: `'ADC'`.

!!! note
    Available only for L card L-502.

---

### digitizer_read_settings() { #digitizer_read_settings }

```python
digitizer_read_settings() -> None
```

!!! example
    `digitizer_read_settings()` reads all the settings of the digitizer.

Reads all the settings from a special text file
[`digitizer.param`](https://github.com/Anatoly1010/Atomize_ITC/tree/master/atomize/control_center)
and writes them to the digitizer using
[`digitizer_setup()`](#digitizer_setup).

!!! note
    Not available for L card L-502.

---

### digitizer_window() { #digitizer_window }

```python
digitizer_window() -> float
```

!!! example
    `digitizer_window()` returns the integration window of the digitizer.

Returns the integration window of the digitizer. The window is used in
[`digitizer_get_curve(integral=True)`](#digitizer_get_curve-integral) and is
set via the
[`digitizer.param`](https://github.com/Anatoly1010/Atomize_ITC/tree/master/atomize/control_center)
text file.

!!! note
    Not available for L card L-502.

---

### digitizer_window_points() { #digitizer_window_points }

```python
digitizer_window_points() -> int
```

!!! example
    `digitizer_window_points()` returns the number of points in the digitizer
    window.

Returns the number of points in the digitizer window. The window is used in
[`digitizer_get_curve(points, phases, ...)`](#digitizer_get_curve-points). The
number of points in the window can be set as the length of the
[`DETECTION` pulse](../pulse_programmer.md#pulser_pulse).

!!! note
    Available only for Insys FM214x3GDA.

---

### digitizer_at_exit(integral=False) { #digitizer_at_exit }

```python
digitizer_at_exit(integral: bool = False) -> numpy.array, numpy.array
```

!!! example
    `digitizer_at_exit()` returns current accumulated arrays from the device
    module.

Similar to
[`digitizer_get_curve(integral=...)`](#digitizer_get_curve-integral), but
always returns the current accumulated arrays — while `digitizer_get_curve`
may return `np.nan` if no new buffer has been transferred from the card to
the computer at the time of execution.

!!! note
    Available only for Insys FM214x3GDA.
