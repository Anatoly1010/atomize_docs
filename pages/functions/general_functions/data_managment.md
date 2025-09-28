---
title: Data Managment
nav_order: 11
layout: page
permlink: /functions/general_functions/data_managments/
parent: General Funtions
grand_parent: Documentation
---

---

To open or save raw experimental data one can use a special module based on Tkinter / PyQt.
Alternatively, it is possible to use the CSV Exporter embedded into Pyqtgraph for saving 1D data and a special option in Liveplot (right click -> Save Data Action) for saving 2D data as comma separated two dimensional numpy array. To call functions from Tkinter module one should create a corresponding class instance:
```python
import atomize.general_modules.csv_opener_saver_tk_kinter as openfile #Tkinter
import atomize.general_modules.csv_opener_saver as openfile #PyQt
file_handler = openfile.Saver_Opener()
```

The following functions are available:
- [open_1D(path, header = 0)](#open_1d)<br/>
- [open_1D_dialog(self, directory = '', header = 0)](#open_1d_dialog)<br/>
- [save_1D_dialog(data, directory = '', header = '')](#save_1d_dialog)<br/>
- [open_2D(path, header = 0)](#open_2d)<br/>
- [open_2D_dialog(directory = '', header = 0)](#open_2d_dialog)<br/>
- [open_2D_appended(path, header = 0, chunk_size = 1)](#open_2d_appended)<br/>
- [open_2D_appended_dialog(directory = '', header = 0, chunk_size = 1)](#open_2d_appended_dialog)<br/>
- [save_2D_dialog(data, directory = '', header = '')](#save_2d_dialog)<br/>
- [create_file_dialog(directory = '')](#create_file_dialog)<br/>
- [create_file_parameters(add_name, directory = '')](#create_file_parameters)<br/>
- [save_header(filename, header = '', mode = 'w')](#save_header)<br/>
- [save_data(filename, data, header = '', mode = 'w')](#save_data)<br/>

---

## open_1D()
```python
open_1D(path, header = 0) -> data_header, numpy.array
```
```
path is a path to file;
header is an integer to specify the number of columns in the file header;
```
Simple function to open a specified file with comma separated values.

---

## open_1D_dialog()
```python
open_1D_dialog(directory = '', header = 0) -> data_header, numpy.array
```
```
directory is a path to preopened directory in the dialog window;
header is an integer to specify the number of columns in the file header;
```
This function returns a dialog to open a file with comma separated values.

---

## save_1D_dialog()
```python
save_1D_dialog(data, directory = '', header = '') -> none
```

```
data is numpy array of data to save;
directory is a path to preopened directory in the dialog window;
header is a string to prepend to a file as a header;
The values will be saved in '%.4e' format;
```

This function returns a dialog to save data as comma separated values.

---

## open_2D()
```python
open_2D(path, header = 0) -> text_header, numpy.array
```

```
path is a path to file;
header is an integer to specify the number of columns in the file header;
```

Simple function to open a specified file with 2D array of comma separated values.

---

## open_2D_dialog()
```python
open_2D_dialog(directory = '', header = 0) -> text_header, numpy.array
```
```
directory is a path to preopened directory in the dialog window;
header is an integer to specify the number of columns in the file header;
```

This function returns a dialog to open a file with 2D array of comma separated values.

---

## open_2D_appended()
```python
open_2D_appended(path, header = 0, chunk_size = 1) -> text_header, numpy.array
```
```
path is a path to file;
header is an integer to specify the number of columns in the file header;
chunk_size is the Y axis size of the initial 2D array;
```

This function returns a dialog to open a file with a single column array of values from 2D array.

---

## open_2D_appended_dialog()
```python
open_2D_appended_dialog(directory = '', header = 0, chunk_size = 1) -> text_header, numpy.array
```
```
directory is a path to preopened directory in the dialog window;
header is an integer to specify the number of columns in the file header;
chunk_size is the Y axis size of the initial 2D array;
```

This function returns a dialog to open a file with a single column array of values from 2D array.

---

## save_2D_dialog()
```python
save_2D_dialog(data, directory = '', header = '') -> none
```
```
data is numpy 2D array of data to save;
directory is a path to preopened directory in the dialog window;
header is a string to prepend to a file as a header;
The values will be saved in '%.4e' format;
```

This function returns a dialog to save a 2D array as comma separated values.

---

## create_file_dialog()
```python
create_file_dialog(directory = '') -> path_to_file.csv
```
```
directory is a path to preopened directory in the dialog window;
```

Function that returns a dialog to enter a name for a file. It can be used to manually saved your data inside the experimental script to specified file.

---

## create_file_parameters()
```python
create_file_parameters(add_name, directory = '') -> path_to_file.add_name, -> path_to_file.csv
```
```
add_name argument is a string that will be added to the second file instead of '.csv' extension.
directory is a path to preopened directory in the dialog window;
Example: create_file_parameters('.param') will create a second file with .param extension;
```

This function has the full functionality of the [create_file_dialog()](#create_file_dialog) function, but also returns a second file for saving parameters / header.

---

## save_header()
```python
save_header(filename, header = '', mode = 'w') -> none
```
This function save the string given by argument header to the file with the path 'filename'. Argument mode allows choosing whether the file will be rewritten (mode = 'w') or the data will be appended to the end of the file (mode = 'a').

---

## save_data()
```python
save_data(filename, data, header = '', mode = 'w') -> none
```
This function save the numpy array given by the argument data and the string given by argument header to the file with the path 'filename'. Argument mode allows choosing whether the file will be rewritten (mode = 'w') or the data will be appended to the end of the file (mode = 'a').<br/>
This function works for 1D, 2D, and 3D data. In case of 3D (an array of 2D arrays) data, a separate file will be created for each 2D array with the additional '_i' string in the filename. The standard combination of function to save the experimental data together with a header is the following:
```python
file_data, file_param = file_handler.create_file_parameters('.param')
header = 'Test Header'
file_handler.save_header(file_param, header = header, mode = 'w')
# Acquiring experimental data
file_handler.save_data(file_data, data, header = header, mode = 'w')
```

---

## Standard numpy savetxt() function
```python
np.savetxt(path_to_file, data_to_save, fmt = '%.4e', delimiter = ' ', 
	newline = 'n', header = 'field: %d' % i, footer = '',
	comments = '#',encoding = None)
```
For saving inside the script by [create_file_dialog()](#create_file_dialog) a standard numpy function should be used.
