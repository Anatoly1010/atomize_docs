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

## 1D plotting in the test run { #1d-plotting-in-the-test-run }

```python
plot_1d_test('name', Xdata, Ydata, label='label', xname='NameXaxis',
        xscale='XaxisDimension', yname='NameYaxis', yscale='YaxisDimension',
        scatter='False', timeaxis='False', vline='False',
        pr='None', text='')
```

Same arguments and parallel-drawing semantics as [`plot_1d()`](#1d-plotting), but the curve is drawn **only when the script is launched in test mode** (i.e. when the GUI's *Test Scripts* checkbox is on, or the script is invoked as `python script.py test`). In a normal experimental run `plot_1d_test()` is a no-op and returns `None`.

Use it to inspect intermediate data — generated waveforms, computed pulse shapes, derived sequences — during the pre-flight check without cluttering plots during real acquisition. It mirrors the [`message_test()`](../general_functions/general_functions.md#print-a-string-in-the-main-window-in-the-test-run) idiom.

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

## Colormap and levels (2D) { #colormap }

The 2D image view offers a choice of colormap and levelling via the image right-click
→ **Colormap** menu. The mode is **Default** unless you pick another; it is remembered
for the rest of the run (per image dock).

| Mode | Use for |
| ---- | ------- |
| **Default** | The standard auto-levelling (data min–max on a blue–white–red map) — the original behaviour, with no extra cost. This is the default |
| **Auto** | Chooses automatically — a **sequential** (viridis) map for one-sided signals, a **bipolar** blue–white–red map for two-sided ones. The first (non-trivial) choice is *locked for the run*, so a live, dynamically re-pushed plot cannot flip colormaps mid-acquisition (it re-decides on a new dataset or when Auto is re-selected) |
| **Bipolar** | Blue–white–red diverging map |
| **Sequential** | Perceptually-uniform viridis map |

Two toggles refine the levelling in the **Auto / Bipolar / Sequential** modes:

- **Center bipolar on baseline** — pins the white of the bipolar map to the estimated
  baseline (the data median) instead of the level midpoint, so a **non-zero baseline
  reads as neutral** and the positive/negative excursions stay directly comparable even
  when they are asymmetric. Recommended for baseline-subtracted or bimodal maps.
- **Per-frame auto-levels (stacks)** — when a multi-frame stack is plotted, level and
  colour **each frame on its own statistics**, so e.g. a raw (non-zero baseline) frame
  and a baseline-subtracted frame in the same stack both display correctly. Turn it off
  to share one fixed scaling across all frames when you want them directly comparable
  (e.g. a relaxation series).

The baseline used by these modes is estimated from a subsample of the data, so enabling
them stays as fast as the default levelling even on large maps. The histogram to the
right of the image can always be dragged to set levels manually.

---

## 2D plotting in the test run { #2d-plotting-in-the-test-run }

```python
plot_2d_test('name', data, start_step=((Xstart, Xstep), (Ystart, Ystep)),
        xname='NameXaxis', xscale='XaxisDimension',
        yname='NameYaxis Field', yscale='YaxisDimension',
        zname='NameZaxis', zscale='ZaxisDimension',
        pr='None', text='')
```

Same arguments and parallel-drawing semantics as [`plot_2d()`](#2d-plotting), but the surface is drawn **only when the script is launched in test mode**. In a normal experimental run `plot_2d_test()` is a no-op and returns `None`. Use it for diagnostic 2D inspection during the pre-flight check without polluting plots during real acquisition.

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
