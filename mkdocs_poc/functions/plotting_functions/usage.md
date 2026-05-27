# Usage

To call plotting functions one should import general function module inside a script:

```python
import atomize.general_modules.general_functions as general
```

## 1D plotting { #1d-plotting }

```python
plot_1d('name', Xdata, Ydata, label='label', xname='NameXaxis',
        xscale='XaxisDimension', yname='NameYaxis', yscale='YaxisDimension',
        scatter='False', timeaxis='False', vline='False',
        pr='None', text='')
```

| Argument          | Description |
| ----------------- | ----------- |
| `Xdata`, `Ydata`  | numpy arrays or python lists of data |
| `name`            | String of the plot name |
| `label`           | String of the curve name |
| `text`            | Label of the plot (for multithreading only) |
| `scatter`         | `['False', 'True']`. Enables scatter plot |
| `timeaxis`        | `['False', 'True']`. Enables time axis |
| `vline`           | `'False'` or a tuple `(vline1, vline2)`. Shows up to two vertical lines |
| `pr`              | Name of process for plotting in another thread |

Other arguments are optional and used for automatic scaling (i.e. V to mV, etc.). Please, note that it is impossible to redraw a line with scatters if they have the same name.

There is a possibility to draw data in parallel with the main script. To do this, a keyargument `pr` should be used. In this mode, the function returns the thread that starts the drawing, and check on the next call whether the process has completed or not. A minimal working example for parallel drawing with dynamic label is the following:

```python
import atomize.general_modules.general_functions as general
process = 'None'
for i in range(10):

    # draw one curve (x_axis, data_x) with a vertical line and a dynamic label "text"
    process = general.plot_1d('NAME_1', x_axis, data_1, xname='Delay',
        xscale='ns', yname='Area', yscale='V*s', label='curve',
        vline=(i, ), pr=process, text=str(i))

    # draw two curves (x_axis, data_x) and (x_axis, data_y) simultaneously
    process = general.plot_1d('NAME_2', x_axis, (data_1, data_2), label='curve',
        xname='Delay', xscale='ns', yname='Area', yscale='V*s',
        vline=(i, ), text=str(i))
```

A tuple `(data_1, data_2)` can be used as `Ydata`. In this case two curves will be plot simultaneously (see example above). A dynamic label based on the `text` keyword argument works only for parallel drawing. For a standard drawing mode a function [`general.text_label()`](#dynamic-labeling) should be used.

---

## 2D plotting { #2d-plotting }

```python
plot_2d('name', data, start_step=((Xstart, Xstep), (Ystart, Ystep)),
        xname='NameXaxis', xscale='XaxisDimension',
        yname='NameYaxis Field', yscale='YaxisDimension',
        zname='NameZaxis', zscale='ZaxisDimension',
        pr='None', text='')
```

| Argument     | Description |
| ------------ | ----------- |
| `data`       | Multidimensional numpy array; example = `[[1,1,1,..],[2,2,2,..],[3,3,3,..],..]` |
| `name`       | String of the plot name |
| `label`      | String of the curve name |
| `text`       | Label of the plot (for multithreading only) |
| `pr`         | Name of process for plotting in another thread |
| `start_step` | List of starting points and steps for the X and Y axis |

Other arguments are optional and used for automatic scaling (i.e. V to mV, etc.).

There is a possibility to draw data in parallel with the main script. To do this, a key argument `pr` should be used. In this mode, the function returns the thread that starts the drawing, and check on the next call whether the process has completed or not. A minimal working example for parallel drawing with dynamic label is the following:

```python
import atomize.general_modules.general_functions as general
process = 'None'
for i in range(10):

    process = general.plot_2d('NAME', data, start_step=((0, 1), (0, 1)),
        xname='Time', xscale='ns', yname='Delay', yscale='ns',
        zname='Intensity', zscale='V', pr=process, text=str(i))
```

---

## Dynamic labeling { #dynamic-labeling }

```python
text_label('label', DynamicValue)
```

| Argument       | Description |
| -------------- | ----------- |
| `label`        | String |
| `DynamicValue` | Number that will change dynamically |

---

## Clearing { #clearing }

```python
plot_remove('name_of_plot')
```
