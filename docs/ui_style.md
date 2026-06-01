# UI Styling

Atomize Qt windows share a single dark theme. This page explains how that theme
is applied and what to do (and not do) when you add or restyle a window.

## Why a shared style is needed

Atomize runs as several independent Qt processes — the main window plus, on the
EPR endstation, each control-center tool runs in its own `QProcess` with its own
`QApplication`. Each used to define its colours inline with `setStyleSheet(...)`.

Those inline sheets only set **foreground** properties (text colour, selection
colour). Everything else — field backgrounds, borders, the spin-box `+/-`
glyphs, the combo arrow — fell through to the platform's *native* Qt style. That
style is **Fusion** on Linux but **windowsvista** on Windows, and the two draw
those parts very differently:

- `QComboBox` — wrong background colour and a border that doesn't match the
  other widgets;
- `QSpinBox` / `QDoubleSpinBox` — missing `+/-` signs, odd border colour;
- `QLineEdit` — a border that doesn't match the theme.

The fix is to pin every process to the **Fusion** style with a shared dark
**`QPalette`**, so the baseline renders identically on both platforms.

## The shared module

All of this lives in one framework-only module:

```
atomize/general_modules/gui_style.py
```

Call `apply_app_style()` once, right after the `QApplication` is created and
**before** any window is shown:

```python
from atomize.general_modules.gui_style import apply_app_style

def main():
    app = QApplication(sys.argv)
    apply_app_style(app, app_id='Atomize.MainWindow')
    window = MainWindow()
    window.show()
    sys.exit(app.exec())
```

`apply_app_style(app, app_id=None, theme=DEFAULT_THEME)` does three things:

1. sets the Fusion style (`QStyleFactory.create('Fusion')`);
2. applies the dark `QPalette` derived from `theme`;
3. on Windows, sets the process AppUserModelID (see
   [Taskbar icon](#windows-taskbar-icon) below) when `app_id` is given.

!!! note "One call per process"
    Because each tool is its own `QApplication`, every entry point must make its
    own `apply_app_style(...)` call. Setting it in one process does not affect
    the others.

## Palette vs. stylesheet: division of labor

After `apply_app_style`, think of the look as two layers:

- **`QPalette` (the baseline)** — backgrounds, borders, default text and
  selection colours, the spin `+/-` glyphs, the combo arrow. This is the layer
  that used to be native and platform-divergent.
- **`setStyleSheet(...)` (the deviations)** — anything the palette cannot
  express, or any colour that intentionally differs from the palette.

So the per-widget stylesheets are **still needed** — but only for genuine
deviations. As a rule of thumb:

| Still needed (deviation)                                   | Now redundant (palette covers it)                 |
| ---------------------------------------------------------- | ------------------------------------------------- |
| Buttons: `border-radius`, pressed-state colour swap        | Spin boxes: `color` + `selection-*` (= palette)   |
| Line edits with gold text (differs from palette `Text`)    | `QMainWindow { background-color: ... }`           |
| Combo **popup** highlight that inverts the global highlight | A bare `color:` that just restates palette `Text` |
| Tabs, scroll bars, check-box indicators, progress bars     |                                                   |
| `QLabel { font-weight: bold }`                             |                                                   |

Redundant sheets are harmless — leave existing ones in place — but new windows
should only style the deviations and let the palette do the rest.

## Reusing the shared sheets

For the common widgets, import the ready-made sheet strings instead of
hand-writing them, so every window stays consistent:

```python
from atomize.general_modules.gui_style import (
    BG, FG, ACCENT, BUTTON_STYLE, LABEL_STYLE, COMBO_STYLE,
    SPIN_STYLE, DSPIN_STYLE, LINEEDIT_STYLE, CHECKBOX_STYLE,
    SCROLL_STYLE, TAB_STYLE,
)

btn.setStyleSheet(BUTTON_STYLE)
combo.setStyleSheet(COMBO_STYLE)
```

## Re-skinning (themes)

The whole look is described by one dataclass of RGB tuples, `Theme`, with
`DEFAULT_THEME` as the standard dark palette. To change the look, build a
custom theme and pass it through:

```python
from atomize.general_modules.gui_style import Theme, apply_app_style, build_styles

my_theme = Theme(accent=(120, 200, 255))     # blue highlight instead of gold
apply_app_style(app, theme=my_theme)

styles = build_styles(my_theme)              # matching per-widget sheets
btn.setStyleSheet(styles['BUTTON_STYLE'])
```

`build_palette(theme)` and `build_styles(theme)` return the palette and the
sheet dictionary for any theme, so the palette and the stylesheets never drift
apart.

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
