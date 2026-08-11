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

## Saving the full 2D data

The two tools that acquire a full 2D array can write it as a single [HDF5](../functions/general_functions/data_managment.md#hdf5-files) file instead of comma separated text. The 1D result of an experiment stays CSV in both of them; only the 2D arrays change format, and reading either format stays supported everywhere.

In the **AWG phasing** tool the Settings tab carries the pair of checkboxes. "Save 2D" saves the full 2D array next to the integrated 1D result, as before; "Save 2D as HDF5" makes every 2D dump of the run an `.h5` file — the `_2d` companion, the primary dump when the phase correction is off (so the 2D array is the result itself), and the per-cycle files of an ESEEM tau average, which stay one file per cycle. The 1D result files are unaffected, and are now written with three more digits than before.

In the **TR EPR** tool a single "Save as HDF5" checkbox picks the format for the whole run: the save dialog then offers `.h5`, and the derived `_osc2` and `_pulse` files follow the name you choose. With "Save Each Scan" on, an HDF5 run no longer writes a whole new `_{j}_scans` file per scan — the cumulative average after each scan is appended as one slice of a `scans` dataset inside the same file, which is both far faster inside the acquisition loop and far smaller on disk.

The main window's "Open 1D Data" / "Open 2D Data" / "Open TR Data" actions and both Data Treatment windows read `.h5` as well as `.csv`. A 2D file holding both quadratures opens as I and Q together, and the axes stored in the file are used to set the plot scales.

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
