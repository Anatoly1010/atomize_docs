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
