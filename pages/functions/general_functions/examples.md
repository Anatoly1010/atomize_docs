---
title: Examples
nav_order: 20
layout: page
permlink: /functions/general_functions/examples/
parent: General Funtions
grand_parent: Documentation
---

---

## Open CSV data
```python
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver_tk_kinter as openfile

file_handler = openfile.Saver_Opener()

head, data = file_handler.open_2D_dialog(header = 0):

general.plot_2d('Plot Z Test', data, start_step = ((0, 1), (0.3, 0.001)),
	xname = 'Time', xscale = 's', yname = 'Magnetic Field', yscale = 'T', 
	zname = 'Intensity', zscale = 'V')
```

---

## Save data
```python
import numpy as np
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver as openfile

file_handler = openfile.Saver_Opener()

data = [];
general.message('Test of saving data')
path_to_file = file_handler.create_file_dialog(directory = '')

## 2D Experiment
for _ in range(10):
	axis_x = np.arange(4000)
	ch_time = np.random.randint(250, 500, 1)
	zs = 1 + 100*np.exp(-axis_x/ch_time) + 7*np.random.normal(size = (4000))
	data.append(zs)
	general.wait('100 ms')	

	general.plot_2d('Plot Z Test', data, start_step = ((0, 1), (0.3, 0.001)), 
		xname = 'Time', xscale = 's', yname = 'Magnetic Field', yscale = 'T', 
		zname = 'Intensity', zscale = 'V')

f = open(path_to_file, 'a')
np.savetxt(f, data, fmt = '%.4e', delimiter = ',', newline = 'n',
	header = 'Header text', footer = '', comments = '#', encoding = None)
f.close()
```

---

## Save data during the script
```python
import numpy as np
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver as openfile

file_handler = openfile.Saver_Opener()

data = [];
general.message('Test of saving data')
path_to_file = file_handler.create_file_dialog(directory = '', fmt = '')

f = open(path_to_file, 'a')
## 2D Experiment
for _ in range(10):
	axis_x = np.arange(4000)
	ch_time = np.random.randint(250, 500, 1)
	zs = 1 + 100*np.exp(-axis_x/ch_time) + 7*np.random.normal(size=(4000))
	data.append(zs)
	general.wait('100 ms')
	
	np.savetxt(f, zs, fmt = '%.4e', delimiter = ' ', newline = 'n', 
		header = 'Header text', footer = '', comments = '#', encoding = None)

	general.plot_2d('Plot Z Test', data, start_step = ((0, 1), (0.3, 0.001)),
		xname = 'Time', xscale = 's', yname = 'Magnetic Field', yscale = 'T',
		zname = 'Intensity', zscale = 'V')

f.close()
```

---

## Conccurency example
```python
import time
import numpy as np
import atomize.general_modules.general_functions as general
import atomize.general_modules.returned_thread as returnThread

prPlot = 'None'
POINTS = 50
STEP = 2

data_x = np.zeros(POINTS)
data_y = np.zeros(POINTS)
x_axis = np.linspace(0, (POINTS - 1)*STEP, num = POINTS) 

for i in range(POINTS):

    data_x[i], data_y[i] = np.random.rand(1)[0], np.random.rand(1)[0]
    
    start_time = time.time()
    
    prWait = returnThread.rThread(target = general.wait, args=('150 ms', ), kwargs={})
    prWait.start()
    # Does not affect elapsed time, as it is less than “150 ms” in the wait function from the prWait
    general.wait('200 ms')

    prPlot = general.plot_1d('EXP1', x_axis, (data_x, data_y), label = 'test2', 
    	xname = 'Delay', xscale = 'ns', yname = 'Area', yscale = 'V*s', 
    	vline = (STEP*i, ), pr = prPlot, text=str(STEP*i))
    prWait.join()

    general.message(str(time.time() - start_time))
```
