---
title: Vector Network Analyzer
nav_order: 36
layout: page
permlink: /functions/vector_network_analyzer/
parent: Documentation
---


### Devices
- Planar **C2220**, **S50024** (Socket); Tested 09/2025


---

### Functions
- [vector_analyzer_name()](#vector_analyzer_name)<br/>
- [vector_analyzer_source_power(*pwr, source = 1)](#vector_analyzer_source_powerpwr-source--1)<br/>
- [vector_analyzer_frequency_center(*frq, channel = 1)](#vector_analyzer_frequency_centerfrq-channel--1)<br/>
- [vector_analyzer_frequency_span(*frq, channel = 1)](#vector_analyzer_frequency_spanfrq-channel--1)<br/>
- [vector_analyzer_points(*pnt, channel = 1)](#vector_analyzer_pointspnt-channel--1)<br/>
- [vector_analyzer_trigger_source(*src)](#vector_analyzer_trigger_sourcesrc)<br/>
- [vector_analyzer_send_trigger()](#vector_analyzer_send_trigger)<br/>
- [vector_analyzer_if_bandwith(*bnd, channel = 1)](#vector_analyzer_if_bandwithbnd-channel--1)<br/>
- [vector_analyzer_trigger_mode(*md, channel = 1)](#vector_analyzer_trigger_modemd-channel--1)<br>
- [vector_analyzer_get_data(s = 'S11', type = 'IQ', channel = 1, data_type = 'COR')](#vector_analyzer_get_datas--S11-type--IQ-channel--1-data_type--COR)<br>
- [vector_analyzer_get_freq_data(channel = 1)](#vector_analyzer_get_freq_datachannel--1)<br>
- [vector_analyzer_get_meas_time(channel = 1)](#vector_analyzer_get_meas_timechannel--1)<br>
- [vector_analyzer_command(command)](#vector_analyzer_commandcommand)<br>
- [vector_analyzer_query(command)](#vector_analyzer_querycommand)<br>

---

### vector_analyzer_name()
```python
vector_analyzer_name() -> str
```
This function returns device name.

---
