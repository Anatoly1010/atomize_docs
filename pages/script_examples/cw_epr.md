---
title: CW EPR
nav_order: 10
layout: page
permlink: /script_examples/cw_epr/
parent: Examples
---
<br/>
Assume that we have built a continuous wave (CW) EPR spectrometer and now would like to realize the ability to record CW EPR spectra under different experimental conditions. Without going into detail, recording the EPR spectrum is quite simple: using a lock-in amplifier, it is necessary to measure the amplitude of the signal from the detector located in the microwave bridge at different values of the external magnetic field. The signal is caused by microwave power absorption by the spin system, observed under resonant conditions. To increase the signal to noise ratio, external magnetic field modulation is typically used, requiring the use of the lock-in detection scheme. To make the setup slightly more advanced, we will also add the possibility to measure the exact frequency of the microwave source using a frequency counter. A schematic representation of the main principles of CW EPR spectroscopy, a scheme of a possible experimental setup, and the experimental flow are shown in the figure:
<br/>
![Figure_3](/atomize_docs/pages/scripts_examples/images/figure_3.png)
<br/>
![Figure_32](/atomize_docs/images/figure_3.png)

---

The first part is the import and initialization of the necessary devices and other auxiliary modules:
```python
# import of lock-in amplifier Stanford Research Systems SR-860,
# magnetic field controller Bruker BH-15,
# frequency counter Agilent 53131a
import atomize.device_modules.SR_860 as sr
import atomize.device_modules.BH_15 as bh
import atomize.device_modules.Agilent_53131a as ag
# import of general-purpose methods and a module for file-handling
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver_tk_kinter as openfile
# import of other libraries
import numpy as np

# initialization of the modules used
file_handler = openfile.Saver_Opener()
ag53131a = ag.Agilent_53131a()
sr860 = sr.SR_860()
bh15 = bh.BH_15()
```

---

After that, all the devices should be properly configured and the remaining experimental parameters defined, such as a range of external magnetic field, arrays for storing the obtained data, etc. Devices are configured using the appropriate methods from the device classes. For the rest, standard Python code can be used:
```python
# setting parameters of the EPR spectra recording
START_FIELD = 3000
END_FIELD = 4000
FIELD_STEP = 1
SCANS = 1

# creating arrays for storing data
points = int( (END_FIELD - START_FIELD) / FIELD_STEP ) + 1
data = np.zeros(points)
x_axis = np.linspace(START_FIELD, END_FIELD, num = points)

# configuring of the devices
bh15.magnet_setup( 100, FIELD_STEP)
field = bh15.magnet_field( START_FIELD )
sr860.lock_in_time_constant( '30 ms' )
sr860.lock_in_sensitivity( '10 mV' )
sr860.lock_in_ref_amplitude( '0.5' )
sr860.lock_in_phase( 159.6 ) 
sr860.lock_in_ref_frequency( 100000 )
ag53131a.freq_counter_digits(8)
ag53131a.freq_counter_stop_mode('Digits')
```

---

Finally, we are ready to measure the spectrum and plot the results. This can be done in one or two Python loops, depending on whether multiple identical scans are required, as shown below. Due to the user-friendly names of the methods of the device modules, the described sequence of actions is understandable to users, even if they do not have significant programming experience. The script usually ends with saving the data.
```python
# main sequence of actions for measuring CW EPR spectra
for j in general.scans(SCANS):

    i = 0
    # setting of the initial magnetic field
    field = bh15.magnet_field( START_FIELD )
    # main measurement cycle
    while field <= END_FIELD:
        
        # getting acquired by the lock-in amplifier data, 
        # corresponding to the current magnetic field
        data[i] = ( data[i] * (j - 1) + sr860.lock_in_get_data() ) / j
        # plotting a graph
        general.plot_1d( 'CW EPR', x_axis, data,
                          xname = 'Field', xscale = 'G',
                          yname = 'Intensity', yscale = 'V', 
                          label = 'Experiment_1', 
                          text = 'Scan / Field: ' + str(j)
                          + ' / ' + str(field) )
        # changing magnetic field
        field = round( (FIELD_STEP + field), 3 )
        bh15.magnet_field( field )

        i += 1

# requesting microwave frequency
mw_freq = ag53131a.freq_counter_frequency('CH3')

# setting the initial magnetic field after the experiment is complete
general.message('Script finished')
field = bh15.magnet_field( START_FIELD )

# data saving
header = 'header for the acquired data'
file_data, file_param = file_handler.create_file_parameters('.param')
file_handler.save_data(file_data, np.c_[x_axis, data],
                       header = header, mode = 'w')
```