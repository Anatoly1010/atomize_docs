# EPR spectroscopy endstation

EPR spectroscopy endstation is a multi-functional setup located at [the Novosibirsk Free Electron Laser Facility](https://ieeexplore.ieee.org/document/7163372). The endstation consists of 3 EPR machines, namely (i) a pulsed X-band EPR; (ii) a continuous wave X-band EPR; (iii) a time-resolved X-band EPR equipped with a Lotis TII Nd:YAG laser. To be continued...

## Hardware

### Pulsed machine

| Component                 | Model                                                                       |
| ------------------------- | --------------------------------------------------------------------------- |
| Pulse generator / ADC / DAC | [Insys FM214x3GDA](../functions/pulse_programmer.md) (312.5 MHz TTL, 2.5 GHz ADC, 1.5 GHz DAC) |
| Magnetic field controller | BH15                                                                        |
| Microwave bridge          | Micran X-band v2                                                            |
| Temperature controller    | Lakeshore 335                                                               |

### CW EPR machine

| Component                 | Model                                  |
| ------------------------- | -------------------------------------- |
| Lock-in amplifier         | Stanford Research SR-850               |
| Frequency counter         | Agilent 53131A                         |
| Temperature controller    | Lakeshore 335                          |
| Magnetic field controller | BH15                                   |

### TR EPR machine

| Component                 | Model                                  |
| ------------------------- | -------------------------------------- |
| Oscilloscope (primary)    | Keysight DSOX3034A                     |
| Oscilloscope (secondary)  | Keysight DSOX2012A                     |
| Frequency counter         | Agilent 53131A                         |
| Temperature controller    | Lakeshore 335                          |
| Magnetic field controller | BH15                                   |

### Other instruments

| Component      | Model    |
| -------------- | -------- |
| NMR gaussmeter | Sibir 1  |

## Automated tune-up and measurement

The pulsed machine can be driven unattended by
[epr_auto](epr_auto/index.md), a YAML protocol runner that ships with the
endstation's [Atomize_ITC](https://github.com/Anatoly1010/Atomize_ITC)
build: it tunes the spectrometer (vane power, working field, integration
window, phase, pulse calibration, repetition rate), runs T2 / T1
measurements with SNR- and time-budget-driven scan counts, judges every
result and records a full run manifest. See the
[epr_auto overview](epr_auto/index.md) for the manual.

## Per-tool working directories

The endstation runs as several independent Qt processes — the main window plus
each control-center tool in its own `QProcess`. Every open/save dialog uses the
[last-opened-directory](../functions/general_functions/last_dir.md) helper so it
reopens where you left off, remembered independently per tool and across
relaunches.

On top of the common `script` and `data` categories, the endstation build adds
one category per preset tool, so each remembers its own folder independently:

| Key | Tool | File type |
| --- | ---- | --------- |
| `cw` | CW EPR control | `*.cw` |
| `tr` | TR EPR control | `*.tr` |
| `tune` | resonator tune preset | `*.tn` |
| `phase` | RECT phasing | `*.phase` |
| `phase_awg` | AWG phasing | `*.phase_awg` |
| `phase_cor` | phase correction | `*.csv` |

The `data` category here also covers the TR data-open dialogs in addition to the
1D / 2D ones.

On the endstation build these `<key>_lastdir.txt` notes are stored under the
repo's `libs/` runtime directory, next to the other runtime-IPC files (these
files are git-ignored), rather than under the per-user config directory used by
plain Atomize.
