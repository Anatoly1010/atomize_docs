---
title: X-band pulsed EPR
nav_order: 10
layout: page
permlink: /projects/xband/
parent: Projects
---
<br/>
Atomize is used to control X-band pulsed EPR spectrometer located in Novosibirsk Institute of Organic Chemistry. Since modern pulsed EPR spectrometers are rather complex multifunctional devices, the standard Atomize UI was extended into a user-friendly control center with special features. The control center includes:
- a simple UI for the digitizer, oscilloscope, temperature controller, and mw bridge
- presets for standard pulse experiments, such as Hahn echo, inversion recovery, three pulse ESEEM, etc.
- interactive pulse tables for the two microwave channels (rectangular pulses, shaped pulses)
- a UI for post-processing experimental data<br/>

The pulse tables incorporate phase cycling of the pulses, which is a great aid for precise tuning of a pulse experiment. Rapid phase cycling enables on-the-fly Fourier transform (FT) of the data which is used extensively to set up acquisitions and monitor the progress of long experiments. The C and C++ backend of the ADC and pulse programmer supports fast updates of experimental data plots or their FTs at refresh rates as high as 100 Hz. The pulse generator module supports non-uniform sampling, which significantly improves time efficiency for many pulse experiments.

---

Used hardware: Spectrum M4i-4450-x8 PCIe digitizer; Spectrum M4i.6631-x8 PCIe DAC; PulseBlaster ESR-PRO-500 PCI pulse generator; BH15 magnetic field controller;  Keysight DSOX3034A oscilloscope; Stanford Research PTC10 temperature controller; Micran X-band microwave bridge.

---

More details: Isaev N.P., Melnikov A.R., Lomanovich K.A., Dugin M.V., Ivanov M.Y., Polovyanenko D.N., Veber S.L., Bowman M.K., Bagryanskaya E.G. "A broadband pulse EPR spectrometer for high-throughput measurements in the X-band." J. Magn. Res. Open. 2022. 14-15. Art. Num. 100092. [DOI](https://doi.org/10.1016/j.jmro.2022.100092)
