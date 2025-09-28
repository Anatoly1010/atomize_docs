---
title: Plotting Funtions
nav_order: 10
layout: page
permlink: /functions/plotting_functions/
parent: Documentation
has_children: true
---
<br/>
The description of available plotting functions in Atomize.

## General information
Plotting of raw experimental data is available via [Liveplot](https://github.com/PhilReinhold/liveplot) library modified according to the aims of the project.
The general idea of the Liveplot is a possibitily to see your data as it comes in to your script, with minimal effort using an appropriate shell to cover verbose syntax.

The [Liveplot](https://github.com/PhilReinhold/liveplot) provides several possibilities for plotting 1D and 2D data. For raw experimental data plotting mainly four of them are applicable. An additional customizability has been added to these functions in comparison with the original Liveplot library. The functions are the following:

- [plot_1d(*args, **kargs)](/pages/functions/plotting_functions/usage/#1d-plotting)<br/>
- [plot_2d(*args, **kargs)](/pages/functions/plotting_functions/usage/#2d-plotting)<br/>
- [text_label('label', DynamicValue)](/pages/functions/plotting_functions/usage/#dynamic-labeling)<br/>
- [plot_remove('name_of_plot')](/pages/functions/plotting_functions/usage/#clearing)<br/>

## GUI features
In addition to the many features of native pyqtgraph widgets Liveplot has:<br/>
- Double click on plots to bring up cross-hair marker<br/>
- Cross-hair displays cross-section cuts for image plots<br/>
- Restore closed plots by double-clicking the name in the plot list<br/>
- Focus on a single plot by maximizing<br/>
- Right click on image plots<br/>
- Toggle histogram & levels scale<br/>
- Enable/disable auto-rescaling of levels when image is updated<br/>

