# Q-band pulsed EPR

Atomize is used to control Q-band pulsed EPR spectrometer located in Novosibirsk Institute of Organic Chemistry. The general implementation is similar to that of the [X-band pulsed machine](xband.md), but a slightly different set of hardware is used. The spectrometer operates in 33.0-35.5 GHz frequency range, has a 200 W solid-state amplifier, a temporal resolution of 4 ns for mw pulses, and a 1.25 GS/s DAC for generating shaped microwave pulses.

## Hardware

| Component                 | Model                                                                       |
| ------------------------- | --------------------------------------------------------------------------- |
| DAC                       | Spectrum M4i.6631-x8 PCIe                                                   |
| Pulse generator           | [Micran PCIe (based on Insys FMC126P)](../functions/pulse_programmer.md)    |
| Magnetic field controller | Homemade                                                                    |
| Oscilloscope              | Keysight DSOX3034A                                                          |
| Temperature controller    | Lakeshore 335                                                               |
| Microwave bridge          | Micran Q-band                                                               |
