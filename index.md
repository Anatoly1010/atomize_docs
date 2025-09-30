---
title: Atomize
layout: page
nav_order: 1
---
<br/>
Atomize is a modular software designed to control a wide range of scientific and industrial instruments, integrate them into a unified multifunctional setup, and automate routine experimental work. The general idea is close to [FSC2 software](http://users.physik.fu-berlin.de/~jtt/fsc2.phtml) developed by Jens Thomas Törring. Remote control of spectrometers is usually carried out using home-written programs, which are often restricted to doing a certain experiment with a specific set of devices. In contrast, the programs like [FSC2](http://users.physik.fu-berlin.de/~jtt/fsc2.phtml) and [Atomize](https://github.com/Anatoly1010/Atomize) are much more flexible, since they are based on a modular approach for communication with instruments and scripting language (EDL in FSC2; Python in Atomize) for data measuring.

Atomize[^1] uses [liveplot library](https://github.com/PhilReinhold/liveplot) based on pyqtgraph as a main graphics library. [Liveplot](https://github.com/PhilReinhold/liveplot) was originally developed by Phil Reinhold. Since then, several improvements have been made to use it in Atomize, and it has been directly embedded into Atomize.

[Python Programming Language](https://www.python.org/) is used inside experimental scripts, which opens up almost unlimited possibilities for raw experimental data treatment. In addition, with PyQt, one can create experimental scripts with a simple graphical interface, allowing users not familiar with Python to use it. Several examples of scripts (with dummy data) are provided in /atomize/tests/ directory, including a GUI script with extended comments inside. Also a variant of the Atomize with GUI Control Window extension can be found [here.](https://github.com/Anatoly1010/Atomize_NIOCH)<br/>

Currently there are more than 200 instrument specific and general functions available for over 27 different devices, including 6 series of devices. If you would like to write a module for the instrument that is not currently available, please, read this [instruction.](https://github.com/Anatoly1010/Atomize/blob/master/atomize/documentation/writing_modules.md)

---

## Status
<br/>
At the moment, Atomize has been tested and is currently used for controlling several EPR spectrometers using a broad range of different devices. The program has been tested on Ubuntu 18.04 LTS, 20.04 LTS, and 22.04 LTS.


---

## General Structure
<br/>
The geenral structure of Atomize can be visualized by the following figure:<br/>
![Figure_1](/atomize_docs/images/figure_1.png)


---


[^1]: Atomize = A + TOM + ize; A stands for Anatoly, main developer; TOMo stands for the International TOMography center, our organization