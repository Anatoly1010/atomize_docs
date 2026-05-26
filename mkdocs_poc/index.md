# Atomize

Atomize is a modular software designed to control a wide range of scientific
and industrial instruments, integrate them into a unified multifunctional
setup, and automate routine experimental work.

!!! note "Proof-of-concept"
    This is an MkDocs Material rendering of the Atomize documentation, set up
    side-by-side with the existing Jekyll site for comparison. Only the
    [Digitizers](functions/digitizer.md) page has been converted so far.

The general idea is close to [FSC2 software](http://users.physik.fu-berlin.de/~jtt/fsc2.phtml)
developed by Jens Thomas Törring. In contrast to home-written control programs
that are restricted to a single experiment, Atomize is based on a modular
approach for communication with instruments and uses Python as the scripting
language for data measurement.

Atomize uses the [liveplot library](https://github.com/PhilReinhold/liveplot)
based on pyqtgraph as its main graphics library.

## Status

Atomize has been tested and is currently used for controlling several EPR
spectrometers using a broad range of different devices. The program has been
tested on Ubuntu 18.04 / 20.04 / 22.04 LTS and Windows 10.

## Citation

Melnikov A, Vedkal A, Ishchenko A, Veber S.
*Atomize: A Modular Software for Control and Automation of Scientific and
Industrial Instruments.*
Journal of Open Research Software, Vol. 13, Issue 1, Article 26.
DOI: <https://doi.org/10.5334/jors.594>
