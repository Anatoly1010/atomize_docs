# Data Management

To open or save raw experimental data one can use a special module. To call functions from the module one should create a corresponding class instance.

```python
import atomize.general_modules.csv_opener_saver as openfile
file_handler = openfile.Saver_Opener()
```

Alternatively, it is possible to use the CSV Exporter embedded into Pyqtgraph for saving 1D data and a special option in Liveplot (right click → Save Data Action) for saving 2D data as comma separated two dimensional numpy array.

## Functions

### open_1d(file_path, header=0) { #open_1d data-toc-label="open_1d" }

```python
open_1d(file_path, header=0)    # -> (data_header, numpy.array)
```

Simple function to open a specified file with comma separated values.

| Argument    | Description |
| ----------- | ----------- |
| `file_path` | Path to file |
| `header`    | Integer specifying the number of columns in the file header |

---

### open_2d(file_path, header=0) { #open_2d data-toc-label="open_2d" }

```python
open_2d(file_path, header=0)    # -> (data_header, numpy.array)
```

Simple function to open a specified file with 2D array of comma separated values.

| Argument    | Description |
| ----------- | ----------- |
| `file_path` | Path to file |
| `header`    | Integer specifying the number of columns in the file header |

---

### open_2d_appended(file_path, header=0, chunk_size=1) { #open_2d_appended data-toc-label="open_2d_appended" }

```python
open_2d_appended(file_path, header=0, chunk_size=1)    # -> (data_header, numpy.array)
```

This function opens a file with a single column array of values from 2D array.

| Argument     | Description |
| ------------ | ----------- |
| `file_path`  | Path to file |
| `header`     | Integer specifying the number of columns in the file header |
| `chunk_size` | Y axis size of the initial 2D array |

---

### open_file_dialog(directory='') { #open_file_dialog data-toc-label="open_file_dialog" }

```python
open_file_dialog(directory='')    # -> path to file.csv
```

This function returns the path to the file selected in the dialog box that opens.

| Argument    | Description |
| ----------- | ----------- |
| `directory` | Path to preopened directory in the dialog window |

---

### create_file_dialog(directory='') { #create_file_dialog data-toc-label="create_file_dialog" }

```python
create_file_dialog(directory='')    # -> path to file.csv
```

This function returns the path to the file specified in the dialog box that opens. It can be used to manually save your data inside the experimental script to specified file.

| Argument    | Description |
| ----------- | ----------- |
| `directory` | Path to preopened directory in the dialog window |

---

### create_file_parameters(add_name, directory='') { #create_file_parameters data-toc-label="create_file_parameters" }

```python
# returns two paths: file with add_name extension and file.csv
create_file_parameters('.param')
```

This function has the full functionality of the [`create_file_dialog()`](#create_file_dialog) function, but also returns a second file for saving parameters / header.

| Argument    | Description |
| ----------- | ----------- |
| `add_name`  | String that will be added to the second file instead of `'.csv'` extension. Example: `create_file_parameters('.param')` will create a second file with `.param` extension |
| `directory` | Path to preopened directory in the dialog window |

---

### save_header(file_path, header='', mode='w') { #save_header data-toc-label="save_header" }

```python
save_header(file_path, header='', mode='w')
```

This function saves the string given by argument `header` to the file with the path `file_path`. Argument `mode` allows choosing whether the file will be rewritten (`mode='w'`) or the data will be appended to the end of the file (`mode='a'`).

---

### save_data(file_path, data, header='', mode='w') { #save_data data-toc-label="save_data" }

```python
save_data(file_path, data, header='', mode='w')
```

This function saves the numpy array given by the argument `data` and the string given by argument `header` to the file with the path `file_path`. Argument `mode` allows choosing whether the file will be rewritten (`mode='w'`) or the data will be appended to the end of the file (`mode='a'`).

This function works for 1D, 2D, and 3D data. In case of 3D (an array of 2D arrays) data, a separate file will be created for each 2D array with the additional `_i` string in the `file_path`. The standard combination of function to save the experimental data together with a header is the following:

```python
file_data, file_param = file_handler.create_file_parameters('.param')
header = 'Test Header'
file_handler.save_header(file_param, header=header, mode='w')
# Acquiring experimental data
file_handler.save_data(file_data, data, header=header, mode='w')
```

---

## Standard numpy savetxt() function

```python
np.savetxt(path_to_file, data_to_save, fmt='%.4e', delimiter=' ',
           newline='n', header='field: %d' % i, footer='',
           comments='#', encoding=None)
```

For saving inside the script by [`create_file_dialog()`](#create_file_dialog) a standard numpy function should be used.

---

# Bruker native formats

A separate, dependency-free reader opens **Bruker EPR native files** directly,
without converting to CSV first. It is numpy-only (no Qt, no extra packages) and
hardware-agnostic, so it works for any Bruker trace — CW spectra, T1/T2
relaxation, ESEEM, DEER, etc.

```python
import atomize.general_modules.bruker_opener as bruker

reader = bruker.Bruker_Opener()
data = reader.open('my_measurement.DSC')
```

Two format families are supported:

- **BES3T** — Xepr / Elexsys (`.DSC` ASCII descriptor + `.DTA` binary, plus
  optional `.XGF`/`.YGF` companions for non-linear axes). The modern pulse-EPR
  format; complex datasets (`IKKF=CPLX`) carry quadrature data and map onto an
  I/Q pair.
- **ESP / WinEPR** — older EMX (`.par` ASCII + `.spc` binary). WinEPR (PC) is
  little-endian float32; ESP (workstation) is big-endian int32. Mostly CW.

## Functions { #bruker_functions data-toc-label="Functions" }

### Bruker_Opener() { #Bruker_Opener data-toc-label="Bruker_Opener" }

```python
reader = bruker.Bruker_Opener()
```

Creates the reader. All methods below are called on it.

### open(path) { #bruker_open data-toc-label="open" }

```python
data = reader.open(path)    # -> dict
```

Opens a Bruker file and dispatches on the extension: `.dsc`/`.dta`/`.xgf`/`.ygf`
→ [`read_bes3t()`](#read_bes3t); `.par`/`.spc` → [`read_winepr()`](#read_winepr).
You may pass **either** member of a pair (the descriptor or the binary) — the
companion file is found automatically. Raises `ValueError` for an unrecognized
extension.

The returned dict:

| Key | Description |
| --- | ----------- |
| `format` | `'BES3T'` or `'ESP/WinEPR'` |
| `ndim` | `1` or `2` |
| `x` | The X (within-trace) abscissa array |
| `x_name`, `x_unit` | X axis name and unit (from the descriptor) |
| `y` | The Y (indirect) abscissa array (2D only) |
| `y_name`, `y_unit` | Y axis name and unit |
| `complex` | `True` if the data carry quadrature (I/Q) |
| `data` | Raw shaped array — `(nx,)` for 1D, `(ny, nx)` for 2D; complex when applicable |
| `channels` | `[(label, ndarray), …]`: `[('real', …), ('imag', …)]` for complex data, else `[('intensity', …)]` |
| `params` | The parsed descriptor as a `{key: value}` dict |

`channels` is the convenient form the Data Treatment tools register; `data` is
the raw array for direct numerical work.

### read_bes3t(path) / read_winepr(path) { #read_bes3t data-toc-label="read_bes3t / read_winepr" }

```python
data = reader.read_bes3t(path)     # BES3T (.DSC/.DTA)
data = reader.read_winepr(path)    # ESP/WinEPR (.par/.spc)
```

The per-format readers behind [`open()`](#bruker_open); both return the same
dict described above. Call them directly when you already know the format,
otherwise prefer [`open()`](#bruker_open).

```python
import atomize.general_modules.bruker_opener as bruker
import atomize.general_modules.general_functions as general

reader = bruker.Bruker_Opener()
res = reader.open('echo_decay.DSC')
if res['complex']:
    re = dict(res['channels'])['real']
    general.plot_1d('Bruker', res['x'], re, xname='Time', xscale=res['x_unit'])
```
