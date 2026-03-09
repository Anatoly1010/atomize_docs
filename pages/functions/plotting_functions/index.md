---
title: Plotting Funtions
nav_order: 10
layout: page
permlink: /functions/plotting_functions/
parent: Documentation
has_children: true
---
<br/>
## General information
Plotting of raw experimental data is available via [Liveplot](https://github.com/PhilReinhold/liveplot) library modified according to the aims of the project. The general idea of the Liveplot is a possibitily to see your data as it comes in to your script, with minimal effort using an appropriate shell to cover verbose syntax.

The [Liveplot](https://github.com/PhilReinhold/liveplot) provides several possibilities for plotting 1D and 2D data. For raw experimental data plotting mainly four of them are applicable. An additional customizability has been added to these functions in comparison with the original Liveplot library. The functions are the following:

- [plot_1d(*args, **kargs)](/atomize_docs/pages/functions/plotting_functions/usage#1d-plotting)<br/>
- [plot_2d(*args, **kargs)](/atomize_docs/pages/functions/plotting_functions/usage#2d-plotting)<br/>
- [text_label('label', DynamicValue)](/atomize_docs/pages/functions/plotting_functions/usage#dynamic-labeling)<br/>
- [plot_remove('name_of_plot')](/atomize_docs/pages/functions/plotting_functions/usage#clearing)<br/>

---

## GUI features

The Liveplot tab the following additional features in the Current Plots dock:
- delete a selected dock with graphs;
- open 1d data in csv multi-column format and plot it in a new graph dock; 
- open 2d data in csv format and plot it in a new graph dock. 

These features are available in the menu that can be opened by a right-click in the Current Plots dock area.<br>

---

The graph docks have the following additional features: 
- Delete: Middle-click a curve name in the legend to remove it from the graph;
- Bring to Front: Left-click a curve name in the legend to move it to the top layer;
- Shift: Drag a curve with the mouse to shift it vertically or horizontally; 
- Scale: Hold 'Ctrl' while dragging a curve to scale it vertically;
- Reset: 'Alt' + Left-click on a curve to reset its shift and scale to their original values;
- Toggle Widgets: Double-click to show/hide the cross-hair widget (for 1D plots) or the cross-section widget (for 2D plots);
- Lock Position: Middle-click to lock the cross-hair or cross-section widget in its current position;

Additional features are available in the menu that can be opened by a right-click in the graph dock area:<br>
- Import Data: Open 1D data in multi-column CSV format and plot it directly in the graph dock; 
- Export Data: Save 2D data from the displayed contour plot;
- Hide Label: The Hide Label combo box can be used to toggle the visibility of labels on the cross-hair or cross-section widgets.

