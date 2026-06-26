# PyScreenPen

A transparent, always-on-top screen annotation tool for Windows. PyScreenPen sits above every other window on your desktop and lets you draw, highlight, add shapes, sticky text, and laser-pointer effects directly on top of whatever you're presenting, teaching, or screen-recording — without taking a screenshot first.

It's built with **PySide6 (Qt for Python)** and is controlled through a custom radial ("pie") tool menu instead of a traditional toolbar, so the canvas underneath stays unobstructed.

---

## Table of Contents

- [Features](#features)
- [Download](#download)
- [Architecture](#architecture)
  - [High-level overview](#high-level-overview)
  - [Project structure](#project-structure)
  - [Core modules](#core-modules)
- [How It Works (Key Methods)](#how-it-works-key-methods)
  - [Single-instance toggling](#single-instance-toggling)
  - [The transparent overlay window](#the-transparent-overlay-window)
  - [Drawing & the ink buffer](#drawing--the-ink-buffer)
  - [Shape recognition](#shape-recognition)
  - [Selection, move, rotate & scale](#selection-move-rotate--scale)
  - [The radial menu](#the-radial-menu)
  - [Global state & persistence](#global-state--persistence)
  - [Stylus / tablet support](#stylus--tablet-support)
- [Installation (for users)](#installation-for-users)
- [Running from Source (for developers)](#running-from-source-for-developers)
- [Building the EXE and Installer](#building-the-exe-and-installer)
  - [1. Build the standalone executable (PyInstaller)](#1-build-the-standalone-executable-pyinstaller)
  - [2. Build the Windows installer (Inno Setup)](#2-build-the-windows-installer-inno-setup)
- [Keyboard Shortcuts](#keyboard-shortcuts)
- [Known Limitations / Roadmap](#known-limitations--roadmap)

---

## Features

- **Freehand drawing** — two independent pens, highlighter, calligraphy nib (angle-aware), spray/airbrush, laser pointer (auto-fades), and a Bezier-style curve tool.
- **Shapes** — line, arrow, rectangle, circle, polygon, star, with optional fill and stroke style (solid/dashed/dotted/dash-dot).
- **Freehand shape recognition** — draw a rough circle/rectangle/polygon and it auto-snaps to a perfect version (powered by OpenCV contour analysis).
- **Selection & editing** — rectangle or lasso select, then move, rotate, scale, lock, duplicate, or delete any stroke, shape, text box, or pasted image.
- **Text tool** with bold/italic styling.
- **Image import** — paste or import an image directly onto the canvas as a movable/scalable object.
- **Board mode** — turn the transparent overlay into an opaque whiteboard/blackboard, with optional grid, lined, dotted, or coordinate-axis backgrounds (fully styleable: spacing, color, opacity, thickness).
- **Radial tool menu** — a collapsible, draggable pie menu (not a docked toolbar) with ~70 sub-pages covering every tool's settings, color swatches, and dial-based sliders (thickness, opacity, spray density, nib angle, etc.).
- **Tablet/stylus support** — pressure-sensitive strokes, automatic pen ↔ eraser switching when you flip a stylus that has an eraser end.
- **Single-instance toggle** — relaunching the app while it's running simply shows/hides it instead of opening a second copy.
- **Configurable global hotkeys** — rebindable via the in-app Settings window, persisted to disk.
- **Undo/redo**, vanishing/timed strokes (for laser-style emphasis), and a "clear canvas" action.

## Download

> No tagged release has been published to this repository yet. Once one is, it will always be available at the link below — GitHub automatically points this at whichever release is marked "latest":
>
> **[⬇ Download the latest release](https://github.com/ssahu200122/PyScreenPen_latest/releases/latest)**
>
> Until a release exists, build the EXE/installer yourself using the [instructions below](#building-the-exe-and-installer), or run the app directly from source.

---

## Architecture

### High-level overview

PyScreenPen is two cooperating top-level windows sitting on a shared, global state object:

```
┌─────────────────────────────────────────────────────────────┐
│                         main.py                              │
│   (single-instance guard via QLocalServer/QLocalSocket)      │
└───────────────┬───────────────────────────┬──────────────────┘
                │                           │
                ▼                           ▼
   ┌─────────────────────────┐   ┌─────────────────────────────┐
   │   Canvas (QWidget)      │   │  DrawboardMenu (QWidget)     │
   │   ui/overlay/canvas.py  │◄─►│  ui/menu/radial_widget.py    │
   │                         │   │                              │
   │  Fullscreen, frameless, │   │  Small, frameless, draggable │
   │  click-through-capable  │   │  radial ("pie") tool menu     │
   │  transparent window.    │   │  built from MenuPage/MenuItem │
   │  Owns all strokes & the │   │  data in ui/menu/menu_models  │
   │  rendered ink buffer.   │   │                              │
   └───────────┬─────────────┘   └──────────────┬───────────────┘
               │                                │
               └───────────────┬────────────────┘
                                ▼
                  ┌───────────────────────────┐
                  │   core/state.py            │
                  │   StateManager (QObject)   │
                  │   - single global "state"  │
                  │     instance                │
                  │   - active tool, color,     │
                  │     thickness, opacity,     │
                  │     board/pattern settings, │
                  │     shortcuts                │
                  │   - emits Qt signals on      │
                  │     every change             │
                  └───────────────────────────┘
```

Both windows import the same module-level `state` singleton from `core/state.py` and communicate **exclusively through Qt signals** — neither window holds a direct reference to the other's internals (aside from `Canvas.menu_ref`, kept only so clicks on the menu don't fall through to the canvas underneath). This keeps tool logic (state) decoupled from both the rendering surface (`Canvas`) and the tool picker UI (`DrawboardMenu`).

### Project structure

```
PyScreenPen_latest/
├── main.py                  # Entry point: single-instance guard, boots Canvas + DrawboardMenu
├── main.spec                # PyInstaller build spec
├── run_pen.bat              # Launch helper (pythonw, no console window)
├── setup_script.iss         # Inno Setup installer script
├── settings.json            # User-editable default shortcuts (copied to %APPDATA% on first run)
├── logo.ico / logo.png      # App icon
├── assets/                  # All toolbar/cursor/menu PNG icons (~70 files)
├── core/
│   └── state.py             # StateManager - the single source of truth for app/tool state
└── ui/
    ├── settings_window.py   # Qt dialog for rebinding shortcuts & editing pattern defaults
    ├── overlay/
    │   └── canvas.py        # The drawing surface: input handling, rendering, strokes, selection
    └── menu/
        ├── menu_models.py   # Declarative data model for every radial menu page (~70 pages)
        └── radial_widget.py # The radial menu widget: hit-testing, painting, animation, dragging
```

### Core modules

| Module | Responsibility |
|---|---|
| `main.py` | Process bootstrap. Ensures only one instance runs (a second launch sends a `SHUTDOWN` message over a local socket and exits). Creates the `Canvas` and `DrawboardMenu`, wires them together, starts the Qt event loop. |
| `core/state.py` | `StateManager(QObject)` — holds active tool, per-tool color/thickness/opacity/style memory, board/pattern configuration, and rebindable keyboard shortcuts. Exposes ~15 Qt `Signal`s (`tool_changed`, `color_changed`, `selection_changed`, `pattern_changed`, etc.) that both UI windows subscribe to. Persists shortcuts + pattern settings to `%APPDATA%\PyScreenPen\settings.json`. |
| `ui/overlay/canvas.py` | The actual drawing surface — a fullscreen, frameless, translucent `QWidget`. Owns the stroke list, the rasterized ink buffer, all mouse/tablet/keyboard event handling, freehand shape recognition (OpenCV), and the selection/transform (move/rotate/scale) system. |
| `ui/menu/menu_models.py` | Pure data: `MenuItem` and `MenuPage` classes describing every slice of every radial menu page (label, icon, highlight/slice color, the action it triggers, and which sub-page it opens). No Qt widgets here — this is just the menu's content tree. |
| `ui/menu/radial_widget.py` | `DrawboardMenu(QWidget)` — renders whatever `menu_models` describes as an actual interactive radial menu: hit-testing wedges/dials, expand/collapse animation, dragging (with screen-edge clamping), tooltips, and per-category slice coloring. |
| `ui/settings_window.py` | A standard `QDialog` for rebinding global shortcuts and editing default pattern (grid/lines/dots) appearance. Talks to the app purely through `state`. |

---

## How It Works (Key Methods)

### Single-instance toggling

`main.py` starts a `QLocalServer` named `PyScreenPen_Toggle_Server`. On launch, it first tries to *connect* to that name as a client:

- **Connection succeeds** → another instance is already running. It sends a `SHUTDOWN` message and exits immediately — this is what makes "launching the app again" behave like a show/hide toggle.
- **Connection fails** → this is the first instance. It becomes the server, listens for future `SHUTDOWN` messages from subsequent launches, and proceeds to build the UI.

### The transparent overlay window

`Canvas` is constructed with:

```python
self.setWindowFlags(Qt.FramelessWindowHint | Qt.WindowStaysOnTopHint | Qt.Tool)
self.setAttribute(Qt.WA_TranslucentBackground)
```

…and is shown full-screen. Because it's always on top and translucent, anything drawn on it appears to float over every other application. The tricky part is **mouse click-through**: when you're actively drawing/selecting you need the window to *catch* clicks, but the rest of the time (e.g. a pen/eraser tool that should only intercept canvas-area input, or simply wanting to use apps underneath) it should let clicks pass through to whatever's behind it.

This is controlled by `Qt.WA_TransparentForMouseEvents`, toggled by `Canvas._refresh_transparency()` and `Canvas.set_tool()`. Both **only call `setAttribute` when the value actually needs to change** — repeatedly flipping this attribute (which maps to the `WS_EX_TRANSPARENT` extended window style on Windows) at high frequency during a drag was found to cause the OS compositor to briefly misroute input to the window underneath, so the guard is a deliberate efficiency *and* correctness fix.

### Drawing & the ink buffer

`Canvas` keeps two parallel representations of your drawing:

1. **`self.strokes`** — a Python list of dicts, one per stroke/shape/text/image, holding its type, `QPainterPath`, color, thickness, style, etc. This is the "source of truth" (used for undo/redo, selection, hit-testing, and re-rendering).
2. **`self.buffer_pixmap`** — a `QPixmap` the size of the screen that the strokes are rasterized into. `paintEvent` just blits this pixmap rather than re-drawing every stroke every frame.

`redraw_buffer()` clears the pixmap and re-draws every stroke from `self.strokes` onto it. This is **only called when the stroke list actually changes** (a stroke finished, deleted, or a transform/move completed) — not on every mouse-move — because re-rasterizing every stroke on every pixel of motion would be far too expensive for an always-on-top full-screen window. During an active drag/rotate/scale, the strokes being manipulated are excluded from the buffer and painted live, directly in `paintEvent`, on top of the (unchanged) buffer underneath.

### Shape recognition

When you draw with a shape tool that supports snapping, `canvas.py` feeds the raw point cloud into OpenCV:

- `cv2.convexHull` + `cv2.arcLength` + `cv2.approxPolyDP` simplify your freehand path down to its dominant corner points.
- Based on the resulting vertex count and geometry, it classifies the shape (rectangle, polygon, etc.) and replaces your rough path with a perfect one — `cv2.minAreaRect`/`cv2.boxPoints` for rectangles, for example.

### Selection, move, rotate & scale

The select tools (`tool_select_rect`, `tool_select_lasso`, `tool_cursor`) all funnel through one private dispatcher, `_handle_input(pos, pressure, event_type, buttons)`, called from mouse, tablet, and (indirectly) keyboard handlers alike, with `event_type` one of `"press" / "move" / "release"`. This keeps mouse and stylus input behaviorally identical.

- **Selecting**: dragging a rectangle or lasso calls `find_selected_strokes()` on release, which tests each stroke's path against the selection region.
- **Moving**: dragging from inside the selection's bounding box sets `is_moving_selection = True`; every move event calls `move_selection(delta)`, which translates each selected stroke's `QPainterPath` (and `QTransform`, for images) directly — no buffer rebuild until you release.
- **Rotate/scale**: dragging a handle sets `transform_mode` to `"rotate"` or `"scale"`. A `QTransform` is built from the angle/scale delta relative to a fixed anchor or center point, and applied to copies of the original (pre-drag) stroke geometry on every move — so repeated small transforms don't compound floating-point error.

### The radial menu

`menu_models.create_menu_structure()` builds a dict of `MenuPage` objects — one of which (`"root"`) is shown initially — each containing a list of `MenuItem`s. An item can either:
- trigger an `action_key` (emitted as a signal the rest of the app listens for), or
- open another `submenu_id` (navigating deeper into the pie, e.g. *Pen → pen settings → color shades*).

`radial_widget.py`'s `DrawboardMenu` is the renderer: it computes wedge angles from the current page's item count, hit-tests mouse position against inner/rim/outer radii to determine which wedge (and which zone — center hub, main wedge, or outer rim) is under the cursor, and animates expansion/collapse via a `QPropertyAnimation` on a custom `expansion` Qt property. Slice fill colors are derived from each item's `highlight_color` (bright, meaningful colors like "Pen 1 = blue" get a tinted slice; neutral white-highlight utility tools stay a flat dark grey), and the currently active tool's slice gets a bright accent ring drawn from that same color.

### Global state & persistence

`core/state.py`'s `StateManager` is instantiated once at import time (`state = StateManager()`) and imported by name everywhere else — effectively a singleton. It owns:
- the active tool ID and a per-tool memory dict (so switching from Pen 1 to Pen 2 and back restores each pen's own color/thickness/opacity),
- board/pattern background configuration,
- the keyboard shortcut map, loaded from / saved to `settings.json` in `%APPDATA%\PyScreenPen\`.

Every mutation emits a Qt signal; `Canvas` and `DrawboardMenu` each connect the signals they care about (e.g. `tool_changed`, `selection_changed`, `pattern_changed`) rather than polling state — this is what keeps the two windows in sync without either one owning the other.

### Stylus / tablet support

`Canvas.tabletEvent()` handles `QTabletEvent.TabletPress / TabletMove / TabletRelease`, converting stylus pressure into variable stroke width and forwarding to the same `_handle_input` dispatcher mouse events use. It also auto-switches to the eraser tool when a stylus's physical eraser end is detected (`QPointingDevice.PointerType.Eraser`) and switches back to your last pen on flipping back.

---

## Installation (for users)

1. Grab the installer from the [latest release](#download) (or build it yourself — see below).
2. Run `PyScreenPen_Setup.exe` and follow the prompts (admin rights required — it installs to Program Files by default).
3. Launch **PyScreenPen** from the Start Menu or desktop shortcut. Launch it again any time to hide/show the overlay.

---

## Running from Source (for developers)

**Requirements:** Windows, Python 3.10+.

```bash
git clone https://github.com/ssahu200122/PyScreenPen_latest.git
cd PyScreenPen_latest
pip install -r requirements.txt
python main.py
```

Dependencies (see `requirements.txt`):

| Package | Used for |
|---|---|
| `PySide6` | Qt bindings — the entire UI, windowing, painting, and event system |
| `opencv-python` | Freehand shape recognition (contour simplification) |
| `numpy` | Point-array math feeding OpenCV |
| `keyboard` | Global hotkeys that work even when the overlay doesn't have OS focus |

---

## Building the EXE and Installer

### 1. Build the standalone executable (PyInstaller)

```bash
pip install pyinstaller
pyinstaller --noconfirm --onedir --windowed --add-data "assets;assets" --icon="logo.ico" main.py
```

- `--onedir` produces a folder (`dist/main/`) containing `main.exe` plus all dependencies — faster startup than `--onefile`, and what `setup_script.iss` expects.
- `--windowed` suppresses the console window (the app is GUI-only).
- `--add-data "assets;assets"` bundles the icon assets folder alongside the executable (Windows path separator `;` — use `:` instead if building on macOS/Linux).
- `--icon="logo.ico"` sets the `.exe`'s file icon.

This repo already includes a matching `main.spec`, so once you've built it the first time you can equivalently run:

```bash
pyinstaller main.spec
```

Output lands in `dist/main/`, with `main.exe` as the entry point.

### 2. Build the Windows installer (Inno Setup)

The repo includes `setup_script.iss`, an [Inno Setup](https://jrsoftware.org/isinfo.php) script already configured to package the PyInstaller output.

1. Install [Inno Setup](https://jrsoftware.org/isdl.php) (free).
2. Make sure `dist/main/` (from the PyInstaller step above) exists in the same folder as `setup_script.iss`.
3. Open `setup_script.iss` in the Inno Setup Compiler and click **Compile** (or run it headlessly: `iscc setup_script.iss`).
4. The finished installer is written to `Output\PyScreenPen_Setup.exe`.

The installer:
- Installs to `Program Files\PyScreenPen` by default (requires admin).
- Adds Start Menu shortcuts and an optional desktop icon.
- Offers to launch the app immediately after install finishes.
- Registers a standard Windows uninstaller.

To publish this as a downloadable release, create a GitHub Release on this repository and attach `PyScreenPen_Setup.exe` as a release asset — the [download link](#download) above will then resolve to it automatically.

---

## Keyboard Shortcuts

Defaults (all rebindable from the in-app Settings window):

| Action | Shortcut |
|---|---|
| Increase tool size | `Ctrl+Shift+{` |
| Decrease tool size | `Ctrl+Shift+}` |
| Toggle eraser | `Alt+B` |
| Toggle select/cursor mode | `Ctrl+Shift+T` |
| Toggle whiteboard/board mode | `Ctrl+Shift+W` |
| Toggle lasso select | `Ctrl+Shift+L` |
| Clear canvas | `Ctrl+Shift+D` |
| Toggle laser pointer | `Ctrl+Shift+H` |
| Toggle overlay visibility | `Ctrl+Shift+G` |
| Quit app | `Ctrl+Q` |

In-canvas (not rebindable): `B` swap pen 1/2, `E` toggle eraser, `H` highlighter, `L` lasso select, `S` cycle shapes, `Space` toggle pan, `Delete` delete selection, `Esc` deselect / cancel.

---

## Known Limitations / Roadmap

- **Windows only** — the single-instance mechanism, `%APPDATA%` settings path, tray/installer tooling, and global-hotkey library are all Windows-specific.
- **Infinite canvas + zoom** — not yet implemented; the canvas is currently a fixed, screen-sized surface.
- **Multi-page canvases** — not yet implemented; everything lives on one drawing surface per session.
