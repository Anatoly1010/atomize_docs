# Math Modules

Helper modules for offline data analysis: least-squares curve fitting, 1D
signal processing (apodization, zero filling, smoothing, baseline subtraction,
echo-centre detection), and DEER/PDS distance-distribution analysis. They take
and return plain NumPy arrays, so a result can be pushed straight to LivePlot
with [`plot_1d()`](../plotting_functions/usage.md) or saved with
[`save_data()`](../general_functions/data_managment.md#save_data).

!!! note "scipy is an optional dependency"
    The fitting routines, Savitzky–Golay smoothing, and the whole DEER engine
    require `scipy`, which is part of the `math` extra:

    ```bash
    pip install -e .[math]
    ```

    The modules import `scipy` lazily, so importing them never fails on a
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

## [DEER / PDS analysis](deer.md)

`import atomize.math_modules.deer as deer`

Distance-distribution analysis for pulsed-dipolar spectroscopy (DEER/PELDOR,
RIDME, DQC, SIFTER): background correction + Tikhonov/NNLS inversion of the
orientation-averaged dipolar kernel, with L-curve regularization. Times in µs,
distances in nm.

| Function | Description |
| -------- | ----------- |
| [`deer_invert(t, V, …)`](deer.md#deer_invert) | One-call pipeline: background-correct → kernel → P(r) |
| [`dipolar_kernel(t, r, …)`](deer.md#dipolar_kernel) | Orientation-averaged kernel K(t, r) (Fresnel closed form) |
| [`dipolar_frequency(r, …)`](deer.md#dipolar_frequency) | Perpendicular dipolar frequency ν⊥(r) = ν_dd/r³ |
| [`background_fit(t, V, bg_start, bg_end=None, …)`](deer.md#background_fit) | Fit intermolecular background on a tail window |
| [`tikhonov_nnls(K, F, alpha, L=None)`](deer.md#tikhonov_nnls) | Non-negative Tikhonov solve K P = F |
| [`regularization_matrix(n, order=2)`](deer.md#regularization_matrix) | Derivative operator L for smoothing |
| [`l_curve(K, F, alphas, L=None)`](deer.md#l_curve) | L-curve scan; corner via Menger curvature |
| [`default_r_axis(rmin=1.5, rmax=8.0, n=200)`](deer.md#default_r_axis) | Default distance grid (nm) |
| [`simulate(t, r, P, …)`](deer.md#simulate) | Forward-simulate a DEER trace from P(r) |
