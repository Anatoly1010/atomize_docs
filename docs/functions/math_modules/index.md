# Math Modules

Helper modules for offline data analysis: least-squares curve fitting and 1D
signal processing (apodization, zero filling, smoothing, baseline subtraction,
echo-centre detection). They take and return plain NumPy arrays, so a result can
be pushed straight to LivePlot with [`plot_1d()`](../plotting_functions/usage.md)
or saved with [`save_data()`](../general_functions/data_managment.md#save_data).

!!! note "scipy is an optional dependency"
    The fitting routines and Savitzky–Golay smoothing require `scipy`, which is
    part of the `math` extra:

    ```bash
    pip install -e .[math]
    ```

    Both modules import `scipy` lazily, so importing them never fails on a
    minimal install — only the functions that need `scipy` raise a
    `RuntimeError` when it is missing.

## [Least-squares fitting](fitting.md)

`import atomize.math_modules.least_square_fitting_modules as math_modules`

| Function | Description |
| -------- | ----------- |
| [`math()`](fitting.md#class) | Create a fitter exposing the model registry |
| [`fit(model, x, y, guess=None, no_offset=False)`](fitting.md#fit) | Fit `(x, y)` with a named model; returns a result dict |
| [`model_names()`](fitting.md#model_names) | List the available model keys |
| [`param_names(model)`](fitting.md#param_names) | Parameter names of a model |
| [`default_guess(model, x, y)`](fitting.md#default_guess) | Heuristic initial guess for a model |
| [`one_exp_fit(curve, guess_array)`](fitting.md#one_exp_fit) | Legacy single-exponential fit |

## [Signal processing](signal_processing.md)

`import atomize.math_modules.signal_processing as sigproc`

| Function | Description |
| -------- | ----------- |
| [`apodization_window(n, name, param=8.6)`](signal_processing.md#apodization_window) | Build an apodization window of length `n` |
| [`zerofill_length(length, choice)`](signal_processing.md#zerofill_length) | Target FFT length for a zero-fill choice |
| [`echo_center(envelope, window=0)`](signal_processing.md#echo_center) | Echo-centre index from a magnitude envelope |
| [`Signal_Processing()`](signal_processing.md#class) | Smoothing / baseline / normalization helpers |
| [`savitzky_golay(y, window=11, order=3)`](signal_processing.md#savitzky_golay) | Savitzky–Golay smoothing (needs scipy) |
| [`moving_average(y, window=5)`](signal_processing.md#moving_average) | Centred moving-average smoothing |
| [`baseline_poly(x, y, order=1, region='all', npts=0)`](signal_processing.md#baseline_poly) | Subtract a polynomial baseline |
| [`normalize(y, mode='minmax')`](signal_processing.md#normalize) | Normalize a curve |
