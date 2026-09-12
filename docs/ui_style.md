# UI Styling

Atomize uses a shared dark theme for the main workspace, plotting menus and control-center tools. Every tool runs in a separate Qt process, so each entry point must apply the theme to its own `QApplication`.

## Apply the theme

The shared implementation is `atomize/general_modules/gui_style.py`. Apply it before creating the window:

```python
from PyQt6.QtWidgets import QApplication
from atomize.general_modules.gui_style import apply_app_style

app = QApplication(sys.argv)
apply_app_style(app, app_id='Atomize.MainWindow')
window = MainWindow()
window.show()
sys.exit(app.exec())
```

`apply_app_style(app, app_id=None, theme=REFINED_THEME, desktop=False)` selects Qt Fusion, installs the theme palette and styles tooltips, menus and separators application-wide. It also supports Windows taskbar identity and Linux desktop integration. Fusion avoids dependence on the native Windows or Linux widget style; visual changes should still be checked on both platforms.

`REFINED_THEME` is the current default. `DEFAULT_THEME` and `build_styles()` remain available for older callers; use `build_refined_styles()` for the current appearance. Changing a palette alone does not override a widget's explicit stylesheet.

## Reuse styles

Use the shared sheets instead of copying colour literals into each window:

```python
from atomize.general_modules.gui_style import (
    REFINED_STYLES, BUTTON_STYLE, COMBO_STYLE, DSPIN_STYLE,
    CHECKBOX_STYLE, ANALYSIS_TAB_STYLE,
)

window.setStyleSheet(REFINED_STYLES['WINDOW_STYLE'])
button.setStyleSheet(BUTTON_STYLE)
combo.setStyleSheet(COMBO_STYLE)
spin.setStyleSheet(DSPIN_STYLE)
check.setStyleSheet(CHECKBOX_STYLE)
analysis_tabs.setStyleSheet(ANALYSIS_TAB_STYLE)
```

When loading a `.ui` file, apply the shared styles after `uic.loadUi()`, because the UI file may contain its own stylesheet. SpinBox, ComboBox, LineEdit and TextEdit share the softer input treatment, including editable controls in menus and file dialogs. `INPUT_STYLE` supplies the application baseline; the dedicated per-widget sheets also include it so local styles cannot restore the old bright fill.

| Element | Shared style / convention |
| --- | --- |
| Window | `WINDOW_STYLE`; indigo background `#202131` |
| Buttons | Distinct fill `#34374F`, ordinary outline `#292B40` |
| Input fields and TextEdit | Muted fill `#26283A`, outline `#292B40`, 2 px corner radius; hover `#30334A`, focus `#2C2F44` with a gold outline |
| Dense numeric fields | `SPIN_STYLE` / `DSPIN_STYLE` / `COMPACT_FIELD_STYLE`; instrument panels generally use 26 px height |
| ComboBox | `COMBO_STYLE`, including the popup background and arrow |
| CheckBox | `CHECKBOX_STYLE`; visible unchecked fill and outline, gold checked indicator |
| RadioButton | `RADIO_STYLE`; round indicator, gold selected state and a visible unchecked outline |
| Main actions | `WORKSPACE_ACTION_STYLE`; compact groups stay together when the window grows |
| Start / Stop | `START_BUTTON_STYLE` / `STOP_BUTTON_STYLE`; yellow/red interaction feedback |
| Running Start | `PRIMARY_BUTTON_STYLE`; solid gold indicates the running state in the main window |
| Plot list and Queue | `PLOT_LIST_STYLE`; compact rows, muted selection with gold text and a thin left mark |
| Plot docks | `DOCK_LABEL_STYLE`, `DOCK_CONTENT_STYLE` and `DOCK_CLOSE_STYLE`; framed header and content |
| Progress | `PROGRESS_STYLE`; light track fill, gold value to the right |
| Instrument tabs | `TAB_STYLE`; the page layouts supply their own spacing |
| Analysis tabs | `ANALYSIS_TAB_STYLE`; adds 8 px side insets and 10 px above the content |
| Group separators | `SEPARATOR_STYLE`, or `gui_forms.hline()` / `vline()`; muted 1 px rules |

## Menus and file dialogs

`MENU_STYLE` is applied at application level, so pyqtgraph context menus and embedded axis controls receive the same dark background. The menu bar keeps its bold labels and bottom accent. `COMBO_STYLE` explicitly styles `QComboBox QAbstractItemView`: a stylesheet on a combo can otherwise leave its popup using platform defaults.

For Qt file pickers, call the shared helper after constructing the dialog and setting its options:

```python
from atomize.general_modules.gui_style import style_file_dialog

style_file_dialog(dialog)
```

The existing picker functions use this helper. File-list selection uses a muted background with gold text, while text selection inside editable fields uses a gold fill. These are separate selection treatments.

## Bundle the glyphs

Keep `check.svg`, `plus.svg`, `minus.svg` and `chevron.svg` beside `gui_style.py`. The styles resolve absolute paths from the module location, so the glyphs continue to work after the application changes its working directory. Include all four assets when porting the shared module or building a distribution. Qt's SVG plugin must be available to render them.

## Custom themes

To customize the palette while retaining the current layout rules, derive a theme and build matching sheets:

```python
from dataclasses import replace
from atomize.general_modules.gui_style import (
    REFINED_THEME, apply_app_style, build_refined_styles,
)

my_theme = replace(REFINED_THEME, accent=(120, 200, 255))
apply_app_style(app, theme=my_theme)
styles = build_refined_styles(my_theme)
button.setStyleSheet(styles['BUTTON_STYLE'])
```

Convenience constants such as `BUTTON_STYLE` are generated from `REFINED_THEME` at import time. Use the returned dictionary for a custom theme. The `input_bg`, `input_hover` and `input_focus` theme fields control the softer input surfaces. Focus takes precedence over hover. Explicit semantic colours, such as Stop feedback and the running-state fill, are defined in the style builder and should be reviewed separately when creating a new colour scheme.

## Windows taskbar icon

On Windows, a windowed Python process is grouped under `python.exe` /
`pythonw.exe` in the taskbar, so it shows the generic Python icon instead of the
icon set with `setWindowIcon(...)`. Windows decides the taskbar icon from the
process *AppUserModelID*, not from the window icon.

Passing a unique `app_id` to `apply_app_style(...)` sets that ID
(`SetCurrentProcessExplicitAppUserModelID`) before the window appears, so each
tool gets its own taskbar button and shows its own icon. Use a stable
reverse-dotted string per tool, e.g. `'Atomize.ITC.DataTreatment1D'`. The call
is a no-op on non-Windows platforms.

!!! tip "Crisp icons"
    A single small PNG can look fuzzy when Windows scales it to large taskbar
    sizes. For sharp results use a multi-resolution `.ico` (16/24/32/48/256 px),
    or add several sizes to the `QIcon` via `addFile`.
