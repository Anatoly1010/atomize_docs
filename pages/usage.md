---
title: Usage
nav_order: 2
layout: page
permlink: /usage/
---

### 1. Installation
<br/>
Install from PyPi:

```bash
pip3 install atomize-py
```

Run GUI from terminal:

```bash
atomize
```

To communicate with Liveplot inside a script the general function module should be imported.

```python
import atomize.general_modules.general_functions as general
general.plot_1d(arguments)
```

The text editor used for editing can be specified in atomize/config.ini. The Telegram bot token and message chat ID can be specified in the same file.

### 2. Setting up general configuration data
<br/>
The /atomize directory contains a general configuration file with the name config.ini. It should be changed at will according to the description below:

```python
[DEFAULT]
# configure the text editor that will opened when Edit is pressed:
editor = subl # Linux
editorW = /path/to/text_editor/on/Windows/  # Windows
# configure the directory that will opened when Open 1D Data or Open 2D Data
# feature is used in Liveplot:
open_dir = /path/to/experimental/data/to/open/
# configure the directory that will be opened when Open Script is pressed:
script_dir = /Atomize/atomize/tests
# configure Telegram bot
telegram_bot_token = 
message_id = 
```

### 3. Using device modules
<br/>
To communicate with a device one should:
1) modify the config file (/atomize/device_modules/config/) of the desired device accordingly. Choose the desired protocol (rs-232, gpib, ethernet, etc.) and correct the settings of the specified protocol in accordance with device settings. A little bit more detailed information about protocol settings can be found [here.](https://github.com/Anatoly1010/Atomize/blob/master/atomize/documentation/protocol_settings.md)
2) import the module or modules in your script and initialize the appropriate class. A class always
has the same name as the module file. Initialization connect the desired device, if the settings are correct.

```python
import atomize.device_modules.Keysight_3000_Xseries as keys
import atomize.device_modules.Lakeshore331 as tc
dsox3034t = keys.Keysight_3000_Xseries()
lakeshore331 = tc.Lakeshore331()
name_oscilloscope = dsox3034t.oscilloscope_name()
temperature = lakeshore331.tc_temperature('CH A')
```

The same idea is valid for plotting and file handling modules.

```python
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver_tk_kinter as openfile
file_handler = openfile.Saver_Opener()
head, data = file_handler.open_1D_dialog(header = 0)
general.plot_1d('1D Plot', data[0], data[1], label = 'test_data', yname = 'Y axis', yscale = 'V')
```

### 4. Experimental scripts
<br/>
Python is used to write an experimental script. Examples (with dummy data) can be found in /atomize/tests/ directory.
