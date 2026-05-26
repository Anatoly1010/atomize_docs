# Usage

## Installation

Atomize can be installed from PyPi:

```bash
pip3 install atomize-py
```

Run GUI from terminal:

```bash
atomize
```

## General Configuration

In the terminal where you launched Atomize, the paths to the configuration files and some other details are displayed as follows:

```ini
SYSTEM = Linux
DATA DIRECTORY = /path/to/experimental/data/to/open/
SCRIPTS DIRECTORY = /path/to/atomize/scripts/
MAIN CONFIG PATH = ~/.config/atomize-py/
DEVICE CONFIG DIRECTORY = ~/.config/atomize-py/device_config/
EDITOR = text editor used for editing scripts
```

The `MAIN CONFIG PATH` shows a path to a general configuration file with the name `main_config.ini`. It should be changed at will according to the description below:

```ini
[DEFAULT]
# configure the text editor that will opened when the
# Edit button is pressed.
# "EDITOR":
editor  = subl                              ; Linux
editorW = /path/to/text_editor/on/Windows/  ; Windows

# configure the directory that will opened when Open 1D Data
# or Open 2D Data feature is used in the Liveplot tab.
# "DATA DIRECTORY":
open_dir = /path/to/experimental/data/to/open/

# configure the directory that will be opened when the
# Open Script button is pressed:
# "SCRIPTS DIRECTORY":
script_dir = /path/to/atomize/scripts/

# configure Telegram bot
telegram_bot_token =
message_id =
```

## Instrument Modules

To communicate with a device one should:

- modify the config file located in `DEVICE CONFIG DIRECTORY` of the desired device accordingly. Choose the desired protocol (rs-232, gpib, ethernet, etc.) and correct the settings of the specified protocol in accordance with device settings. A little bit more detailed information about protocol settings can be found [here](protocol_settings.md).
- import the module or modules in your script and initialize the appropriate class. A class always has the same name as the module file. Initialization connect the desired device, if the settings are correct. For more detailed information, see [documentation](functions/digitizer.md).

```python
# importing of the instruments
import atomize.device_modules.Keysight_3000_Xseries as keys
import atomize.device_modules.Lakeshore331 as tc

# initialization of the instruments
dsox3034t    = keys.Keysight_3000_Xseries()
lakeshore331 = tc.Lakeshore331()

# using the instruments
name_oscilloscope = dsox3034t.oscilloscope_name()
temperature       = lakeshore331.tc_temperature('CH A')
```

The same idea is valid for plotting and file handling modules.

```python
# importing of the general purpose modules
import atomize.general_modules.general_functions as general
import atomize.general_modules.csv_opener_saver as openfile

# initialization
file_handler  = openfile.Saver_Opener()
file_data     = file_handler.open_file_dialog()
header, data  = file_handler.open_1d(file_data, header=0)

# using
general.plot_1d('1D Plot', data[0], data[1],
                label='test_data', yname='Y axis', yscale='V')
```

## Experimental Scripts

Python is used to write an experimental script. Examples can be found in the `SCRIPTS DIRECTORY`.

## Additional Interactivity

The Main tab has the following additional features in the Output dock (available via right-click menu):

- clear all text from the dock;
- open the local directory with the device configuration files;
- print in the terminal all available instruments connected to your computer using pyvisa.

Keyboard shortcuts for the buttons on the Main tab — format `Alt + key`:

| Shortcut  | Button     |
| --------- | ---------- |
| `Alt + O` | Open       |
| `Alt + E` | Edit       |
| `Alt + U` | Run        |
| `Alt + T` | Test       |
| `Alt + S` | Stop       |
| `Alt + Q` | Queue      |
| `Alt + H` | Help       |

The Script Editor dock has the following shortcuts:

| Shortcut   | Action                                                     |
| ---------- | ---------------------------------------------------------- |
| `Ctrl + G` | jump to specified line                                     |
| `Ctrl + F` | search for the specified text                              |
| `Ctrl + N` | show the next occurrence                                   |
| —          | hidden characters can be displayed by selecting any text   |

The Queue dock is used to create an execution queue by pressing `Add to Queue` button. All scripts will be [tested](writing_modules.md#test-run) before being added to the queue. The execution order is from top to bottom. If there are items in the queue, pressing the `Stop Experiment` button will stop the execution of the current script and clear the queue. Additional features (available via right-click menu in the Queue dock):

- delete the selected file from the queue;
- clear the queue;
- drag and drop files to change the execution order.

The Liveplot tab has the following additional features in the Current Plots dock (available via right-click menu):

- delete a selected dock with graphs;
- open 1d data in csv multi-column format and plot it in a new graph dock;
- open 2d data in csv format and plot it in a new graph dock.

The graph docks have the following additional features:

| Feature           | Action                                                                                                                                                                                                                                                                                                                                                                                                          |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Delete**        | Middle-click a curve name in the legend to remove it from the graph                                                                                                                                                                                                                                                                                                                                             |
| **Bring to Front** | Left-click a curve name in the legend to move it to the top layer                                                                                                                                                                                                                                                                                                                                              |
| **Shift**         | Drag a curve with the mouse to shift it vertically or horizontally                                                                                                                                                                                                                                                                                                                                              |
| **Scale**         | Hold `Ctrl` while dragging a curve to scale it vertically                                                                                                                                                                                                                                                                                                                                                       |
| **Reset**         | `Alt` + Left-click on a curve to reset its shift and scale to their original values                                                                                                                                                                                                                                                                                                                             |
| **Toggle Widgets** | Double-click to show/hide the cross-hair widget (for 1D plots) or the cross-section widget (for 2D plots)                                                                                                                                                                                                                                                                                                      |
| **Lock Position** | Middle-click to lock the cross-hair or cross-section widget in its current position                                                                                                                                                                                                                                                                                                                             |
| **Ruler**         | Hold `Shift` and left-drag to measure distance between two points. For 1D plots the endpoints snap to the nearest data point of the curve under the press position, and the ruler adopts that curve's color; the label shows ΔX, ΔY, and the (X, Y) of both endpoints. For 2D plots the endpoints snap to the nearest pixel center, and the label shows ΔX, ΔY, ΔZ, and the (X, Y, Z) of both endpoints. Hold `Shift + Ctrl` while dragging to use free coordinates without snapping. The ruler is removed automatically when the cross-hair or cross-section widget is hidden. |

Additional features available via right-click menu in the graph dock area:

| Feature          | Action                                                                                            |
| ---------------- | ------------------------------------------------------------------------------------------------- |
| **Import Data**  | Open 1D data in multi-column CSV format and plot it directly in the graph dock                    |
| **Export Data**  | Save 2D data from the displayed contour plot                                                      |
| **Hide Label**   | Toggle the visibility of labels on the cross-hair or cross-section widgets                        |
| **Clear Ruler**  | Removes the current ruler overlay from the 1D or 2D plot                                          |
