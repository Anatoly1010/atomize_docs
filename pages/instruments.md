---
title: Instruments
nav_order: 4
layout: page
permlink: /instruments/
---
<br>

- [Arbitrary Waveform Generators](#arbitrary-waveform-generators)
- [Delay Generators](#delay-generators)
- [Digitizers](#digitizers)
- [Frequency Counters](#frequency-counters)
- [Gaussmeters](#gaussmeters)
- [Laser Power Meters](#laser-power-meters)
- [Lock-in Amplifiers](#lock-in-amplifiers)
- [Magnetic Field Controllers](#magnetic-field-controllers)
- [Magnet Power Supplies](#magnet-power-supplies)
- [Microwave Bridge Controllers](#microwave-bridge-controllers)
- [Moisture Meters](#moisture-meters)
- [Oscilloscopes](#oscilloscopes)
- [Power Supplies](#power-supplies)
- [Pulse Programmers](#pulse-programmers)
- [Synthetizers](#synthetizers)
- [Temperature Controllers](#temperature-controllers)
- [Vector Network Analyzers](#vector-network-analyzers)
- [Waveform Generators](#waveform-generators)
- [Other](#other)

---

## [Arbitrary Waveform Generators](/atomize_docs/pages/functions/awg)
- Spectrum M4I 6631 X8; Tested 07/2021
The original [library](https://spectrum-instrumentation.com/en/m4i6631-x8) was written by Spectrum. 
- [Insys FM214x3GDA](https://www.insys.ru/mezzanine/fm214x3gda) as DAC; Tested 03/2025
The device is available via ctypes. The original library can be found [here](https://github.com/Anatoly1010/Atomize_ITC/tree/master/libs).

---

## [Delay Generators](/atomize_docs/pages/functions/delay_generator)
- Stanford Research DG535 (GPIB); Untested

---

## [Digitizers](/atomize_docs/pages/functions/digitizer)
- Spectrum M4I 4450 X8; Tested 08/2021
The original [library](https://spectrum-instrumentation.com/en/m4i4450-x8) was written by Spectrum.
- Spectrum M4I 2211 X8; Tested 01/2021
The original [library](https://spectrum-instrumentation.com/en/m4i4450-x8) was written by Spectrum.
- [Insys FM214x3GDA](https://www.insys.ru/mezzanine/fm214x3gda) as ADC; Tested 03/2025
The device is available via ctypes. The original library can be found [here](https://github.com/Anatoly1010/Atomize_ITC/tree/master/libs).
- L card L-502 as ADC; Tested 03/2022
The L card device is available via ctypes. The original library can be found [here](https://www.lcard.ru/products/boards/l-502?qt-ltab=6#qt-ltab).

---

## [Frequency Counters](/atomize_docs/pages/functions/freq_counter)
- Agilent Frequency Counter (GPIB, RS-232)
	53181A; 53131A/132A; Tested 02/2021
	5343A; GPIB, Tested 02/2023
- Keysight Frequency Counter (GPIB, RS-232, Ethernet)
	53230A/220A; Untested

---

## [Gaussmeters](/atomize_docs/pages/functions/gaussmeter)
- Lakeshore 455 DSP (RS-232); Tested 01/2021
- NMR Gaussmeter Sibir 1 (UDP/IP Socket); Tested 04/2024

---

## [Laser Power Meters](/atomize_docs/pages/functions/laser_power_meter)
- Gentec-EO (RS-232)
	Solo2; Tested 12/2025

---

## [Lock-in Amplifiers](/atomize_docs/pages/functions/lock_in)
- Stanford Research Lock-In Amplifier (GPIB, RS-232)
	SR-810; SR-830; SR-850; Tested 02/2021; SR-844; Untested
- Stanford Research Lock-In Amplifier (GPIB, RS-232, Ethernet)
	SR-860; SR-865a; Tested 01/2021

---

## [Magnetic Field Controllers](/atomize_docs/pages/functions/magnet)
- Bruker BH15 (GPIB); Tested 01/2021
- Bruker ER032M (GPIB); Available via BH15 module
- Bruker ER031M (RS-232 using arduino emulated keyboard); Tested 01/2021
- [Homemade](https://patents.google.com/patent/RU2799103C1/en?oq=RU2799103C1) magnetic field controller (RS-232); Tested 04/2023

---

## [Magnet Power Supplies](/atomize_docs/pages/functions/magnet_power_supply)
- Cryomagnetics 4G (Ethernet); Tested 11/2023

---

## [Microwave Bridge Controllers](/atomize_docs/pages/functions/microwave_bridge)
- Micran X-band MW Bridge (TCP/IP Socket); Tested 06/2021
- Micran X-band MW Bridge v2 (TCP/IP Socket); Tested 12/2022
- Micran Q-band MW Bridge (TCP/IP Socket); Tested 12/2023

---

## [Moisture Meters](/atomize_docs/pages/functions/moisture_meter)
- IVG-1/1 (RS-485); Tested 02/2023

---

## [Oscilloscopes](/atomize_docs/pages/functions/oscilloscope)
- Keysight InfiniiVision 2000 X-Series (Ethernet); Tested 07/2021
- Keysight InfiniiVision 3000 X-Series (Ethernet); Tested 06/2021
- Keysight InfiniiVision 4000 X-Series (Ethernet); Untested
- Tektronix 3000 Series (Ethernet); Tested 09/2022
- Tektronix 4000 Series (Ethernet); Tested 01/2021
- Tektronix 5 Series MSO (Ethernet); Tested 12/2023
- Rigol MSO8000 Series (Ethernet); Untested

---

## [Power Supplies](/atomize_docs/pages/functions/power_supply)
- Rigol DP800 Series (RS-232, Ethernet); Tested 01/2021
- Stanford Research DC205 (RS-232); Untested
- Stanford Research PS300 High Voltage Series (RS-232, GPIB); Untested

---

## [Pulse Programmers](/atomize_docs/pages/functions/pulse_programmer)
- Pulse Blaster ESR 500 Pro; Tested 06/2021
    The device is available via ctypes. The original [C library](http://www.spincore.com/support/spinapi/using_spin_api_pb.shtml) was written by SpinCore Technologies.
- Pulse  Programmer Micran based on [Insys FMC126P](https://www.insys.ru/fmc/fmc126p); Tested 12/2023
- [Insys FM214x3GDA](https://www.insys.ru/mezzanine/fm214x3gda) as multichannel TTL pulse generator; Tested 03/2025
The Insys device is available via ctypes. The original library can be found [here](https://github.com/Anatoly1010/Atomize_ITC/tree/master/libs).

---

## [Synthetizers](/atomize_docs/pages/functions/synthetizer)
- ECC 15K (RS-232); Tested 01/2023

---

## [Temperature Controllers](/atomize_docs/pages/functions/temp_controller)
- Lakeshore (GPIB, RS-232)
	325; 331; 332; 335; 336; 340; Tested 01/2021
- Oxford Instruments (RS-232)
	ITC 503; Tested 01/2021
- Termodat (RS-485)
    11M6; 13KX3; Tested 04/2021
- Stanford Research (TCP/IP Socket)
	PTC10; Tested 07/2021
- Scientific Instruments (TCP/IP Socket, RS-232)
	SCM10; Tested 07/2022

---

## [Vector Network Analyzers](/atomize_docs/pages/functions/vector_network_analyzer)
- Planar C2220, S50024 (Socket); Tested 09/2025

---

## [Waveform Generators](/atomize_docs/pages/functions/wave_generator)
- Waveform Generator of Keysight InfiniiVision 2000 X-Series (Ethernet)
	Available via corresponding oscilloscope module.
- Waveform Generator of Keysight InfiniiVision 3000 X-Series (Ethernet)
	Available via corresponding oscilloscope module.
- Waveform Generator of Keysight InfiniiVision 4000 X-Series (Ethernet)
	Available via corresponding oscilloscope module.
- Waveform Generator of Rigol MSO8000 Series (Ethernet)
	Available via corresponding oscilloscope module.
- Stanford Research DS345 (RS-232); Untested.


---

## [Other](/atomize_docs/pages/functions/other_device)
- CPWplus 150 (RS-232); Tested 01/2021
- RODOS-10N Solid-State Relay (Ethernet); Tested 01/2021
- Owen-MK110-220.4DN.4R Discrete IO Module (RS-485); Tested 04/2021
- Cryomagnetics LM-510 Liquid Cryogen Monitor (TCP/IP Socket); Tested 07/2022
- Cryomech CPA2896, CPA1110 Digital Panels (RS-485); Tested 07/2022

