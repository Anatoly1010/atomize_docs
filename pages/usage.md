---
title: Usage
nav_order: 2
layout: page
permlink: /usage/
---
<br>

- [Installation](#installation)<br>
- [General Configuration](#general-configuration)<br>
- [Instrument Modules](#instrument-modules)<br>
- [Experimental Scripts](#experimental-scripts)<br>
- [Additional Interactivity](#additional-interactivity)<br>

---

## Installation
<br/>
Atomize can be installed from PyPi:

```bash
pip3 install atomize-py
```

Run GUI from terminal:

```bash
atomize
```

---

## General Configuration
<br/>
In the terminal where you launched Atomize, the paths to the configuration files and some other details are displayed as follows:

```yml
SYSTEM: Linux
DATA DIRECTORY: /path/to/experimental/data/to/open/
SCRIPTS DIRECTORY: /path/to/atomize/scripts/
MAIN CONFIG PATH: ~/.config/atomize-py/
DEVICE CONFIG DIRECTORY: ~/.config/atomize-py/device_config/
EDITOR: text editor used for editing scripts
```

The "MAIN CONFIG PATH" shows a path to a general configuration file with the name main_config.ini. It should be changed at will according to the description below:

```yml
[DEFAULT]
# configure the text editor that will opened when the Edit  button is pressed.
# "EDITOR":
editor = subl # Linux
editorW = /path/to/text_editor/on/Windows/  # Windows

# configure the directory that will opened when Open 1D Data or Open 2D Data
# feature is used in the Liveplot tab. 
# "DATA DIRECTORY":
open_dir = /path/to/experimental/data/to/open/

# configure the directory that will be opened when the Open Script button is pressed:
# "SCRIPTS DIRECTORY":
script_dir = /path/to/atomize/scripts/

# configure Telegram bot
telegram_bot_token = 
message_id = 
```

---

## Instrument Modules
<br/>
To communicate with a device one should:<br/>
- modify the config file located in "DEVICE CONFIG DIRECTORY" of the desired device accordingly. Choose the desired protocol (rs-232, gpib, ethernet, etc.) and correct the settings of the specified protocol in accordance with device settings. A little bit more detailed information about protocol settings can be found [here.](/atomize_docs/pages/protocol_settings)
- import the module or modules in your script and initialize the appropriate class. A class always has the same name as the module file. Initialization connect the desired device, if the settings are correct. For more detailed information, see [documentation.](/atomize_docs/functions)

```python
# importing of the instruments
import atomize.device_modules.Keysight_3000_Xseries as keys
import atomize.device_modules.Lakeshore331 as tc

# initialization of the instruments
dsox3034t = keys.Keysight_3000_Xseries()
lakeshore331 = tc.Lakeshore331()

# using the instruments
name_oscilloscope = dsox3034t.oscilloscope_name()
temperature = lakeshore331.tc_temperature('CH A')
```

The same idea is valid for plotting and file handling modules.

```python
# importing of the general purpose modules
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver_tk_kinter as openfile

# initialization
file_handler = openfile.Saver_Opener()
head, data = file_handler.open_1D_dialog(header = 0)

# using
general.plot_1d('1D Plot', data[0], data[1], label = 'test_data', yname = 'Y axis', yscale = 'V')
```

---

## Experimental Scripts
<br/>
Python is used to write an experimental script. Examples can be found in the "SCRIPTS DIRECTORY".

---

## Additional Interactivity
<br/>
The Main tab has the following additional features in the Output dock:
- clear all text from the dock; 
- open the local directory with the device configuration files;
- print in the terminal all available instruments connected to your computer using pyvisa.

These features are available from the menu that can be opened by a right-click in the Output dock area.<br>

---

The Liveplot tab the following additional features in the Current Plots dock:
- stop the execution of the experimental script; 
- delete a selected dock with graphs;
- open 1d data in csv multi-column format and plot it in a new graph dock; 
- open 2d data in csv format and plot it in a new graph dock. 

These features are available from the menu that can be opened by a right-click in the Current Plots dock area.<br>

---

The graph docks have the following additional features: 
- delete the selected curve from the graph dock;
- vertically shift the selected curve from the graph dock; 
- open 1d data in csv multi-column format and plot it in this graph dock; 
- save 2d data from the displayed contour plot. 

These features are available from the Plot Options menu of pyqtgraph that can be opened by a right-click in the graph dock area.<br>