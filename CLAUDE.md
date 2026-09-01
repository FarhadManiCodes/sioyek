# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project goal for this checkout

This is a **custom local build** of sioyek targeted at:
- **Host:** Arch Linux on AMD Ryzen 7 PRO 7840U (Zen 4 / `znver4`).
- **Display server:** Wayland only — X11/XWayland fallback not required.
- **Distribution:** single user, single machine; no need for portable binary or AppImage layout.

Sioyek itself has **no direct X11 or Wayland code** — all display interaction is mediated by Qt. "Wayland-only" therefore means selecting the Qt Wayland QPA plugin at runtime; the C++ source does not need patching for it. The real customization surfaces are: CPU/optimization flags, system vs. bundled libraries, optional Qt modules, and the bundled MuPDF build.

### Locked-in build decisions

- **MuPDF:** bundled submodule (`mupdf/`), built once with `znver4` flags and used for **both** sioyek (static link) **and** the user's system tools (`mutool` installed to `~/.local/bin/`). Don't install Arch's `mupdf` package — it would collide with the local one. mupdf's GUI viewers (`mupdf-gl`, `mupdf-x11`) are intentionally **not built** (no GLFW / X11 deps).
- **Optional Qt modules dropped:** `TextToSpeech` only, via the `SIOYEK_NO_TTS` CMake option (guards in `utils.h`, `utils.cpp`, `main_widget.cpp` + `CMakeLists.txt` — see `PERSONAL_PATCHES.md`). `QuickWidgets` + QML touch UI is kept compiled-in but dormant — TOUCH_MODE defaults false, the dead code costs nothing at runtime and dropping it would require a large multi-file patch to maintain across upstream merges.
- **Network:** kept. `QLocalSocket`/`QLocalServer` (used by `RunGuard` for single-instance IPC) live in `Qt::Network` in Qt 6 — dropping the module would break multi-PDF window handoff. Outbound HTTP (paper download, JS extension API) is gated on user action; update check is config-off by default.
- **Install layout:** Portable. Build artifacts and runtime assets all go under `~/.local/share/sioyek/`; a tiny wrapper script at `~/.local/bin/sioyek` exec's the binary; a hand-written `~/.local/share/applications/sioyek.desktop` handles desktop launcher integration. No `make install`, no `LINUX_STANDARD_PATHS`, no sudo. User config naturally goes to `~/.config/sioyek/`, user data to `~/.local/share/sioyek/`.
- **Compiler flags:** `-march=znver4 -O3 -flto=auto -pipe -fno-plt` for both sioyek and mupdf. Link: `-flto=auto` (via CXXFLAGS) plus `-Wl,-O2 -Wl,--as-needed` and `-fvisibility=hidden -fvisibility-inlines-hidden`, applied as `target_*_options(sioyek PRIVATE ...)` in `CMakeLists.txt` (not env `LDFLAGS`).
- **Wayland:** runtime selection via `QT_QPA_PLATFORM=wayland` (or auto-detected when `qt6-wayland` is installed in a Wayland session).

### Required Arch packages

`qt6-base qt6-svg qt6-declarative qt6-wayland harfbuzz sqlite zlib cmake gcc pkg-config`

(Notably **not** installed: `qt6-speech` (dropped via `SIOYEK_NO_TTS`), `mupdf` (bundled and installed locally).)

## Updating from upstream ("check the sioyek upstream and update")

Authoritative runbook: **`UPDATING.md`** (pull → rebuild → install steps) and
**`PERSONAL_PATCHES.md`** (what this `personal` branch changes on top of
`upstream/development` and where merges conflict). Read both before starting.

Short version:

```bash
git fetch upstream
git log --oneline --no-merges HEAD..upstream/development   # review what's new
git merge upstream/development                             # onto 'personal'
```

Then **verify every personal patch survived** (see `PERSONAL_PATCHES.md` for the
full list) — quick check:

```bash
grep -n "SIOYEK_NO_TTS\|SQLite3::SQLite3\|mupdf/build/release\|LINUX_STANDARD_PATHS" CMakeLists.txt
grep -n "SIOYEK_NO_TTS" pdf_viewer/utils.h pdf_viewer/utils.cpp pdf_viewer/main_widget.cpp
grep -n 'global_font_family = "JetBrains Mono"' pdf_viewer/main.cpp
grep -n "embedding.npy\|JetBrainsMono" resources.qrc   # expect NO matches (we removed them)
```

Rebuild + install (no MuPDF rebuild unless the submodule pointer moved):

```bash
cmake --build build-cmake -j$(nproc)
cp build-cmake/sioyek ~/.local/share/sioyek/sioyek
```

Sanity: `ldd ~/.local/share/sioyek/sioyek | grep -i speech` → nothing;
`~/.local/bin/sioyek --help` runs. Confirm the compile line still carries
`-march=znver4 -O3 -flto=auto` (`grep -m1 march build-cmake/compile_commands.json`).

## Build System

Sioyek ships **two parallel build systems**: qmake (`pdf_viewer_build_config.pro`, used by `build_linux.sh`) and CMake (`CMakeLists.txt`). For a custom local build, CMake is preferred — it has `compile_commands.json` export and is easier to drive with custom `CXXFLAGS`. qmake is what upstream CI uses.

### Linux build (qmake — upstream path)

```bash
# 1. Build mupdf first (bundled submodule under mupdf/)
cd mupdf && make USE_SYSTEM_HARFBUZZ=yes -j$(nproc) && cd ..

# 2. Build sioyek (defines linux_app_image — portable layout)
./build_linux.sh
# Output: build/sioyek + adjacent prefs.config, keys.config, shaders/, tutorial.pdf
```

`build_linux.sh` passes `CONFIG+=linux_app_image`, which **statically links the bundled mupdf** from `mupdf/build/release/`. Removing that flag enables `NON_PORTABLE` + `LINUX_STANDARD_PATHS` and links system libs (`-lmupdf -lgumbo -lfreetype -ljbig2dec -ljpeg -lmujs -lopenjp2`) — required when system mupdf is used.

### Linux build (CMake — recommended for this checkout)

```bash
cmake -B build-cmake -DCMAKE_BUILD_TYPE=Release \
      -DSIOYEK_NO_TTS=ON \
      -DCMAKE_CXX_FLAGS="-march=znver4 -O3 -flto=auto -pipe -fno-plt"
cmake --build build-cmake -j$(nproc)
```

Note: `CMakeLists.txt` has a `string(REPLACE "-mno-direct-extern-access" "" CMAKE_CXX_FLAGS ...)` on non-Apple Linux (an old Qt 6 LTO-interop workaround). It is currently a **no-op** — Qt 6 injects that flag through its imported-target interface, not `CMAKE_CXX_FLAGS`, so the flag still shows in every compile line (`grep -c mno-direct build-cmake/compile_commands.json`). Harmless: LTO builds work with it present. It still expects bundled mupdf at `mupdf/build/release/libmupdf.a` and friends, **so step 1 of the qmake recipe (`cd mupdf && make ...`) has to be run first** if `mupdf/build/` is empty (see submodule state below) — unless `CMakeLists.txt` is patched to use system mupdf.

### Recommended custom-build flags

For a `znver4`-targeted release binary:

```bash
export CFLAGS="-march=znver4 -O3 -flto=auto -pipe -fno-plt"
export CXXFLAGS="$CFLAGS"
export LDFLAGS="-Wl,-O2 -Wl,--as-needed -flto=auto"
```

Apply the same flags when building mupdf:

```bash
cd mupdf
make XCFLAGS="-march=znver4 -O3 -flto=auto" USE_SYSTEM_HARFBUZZ=yes \
     USE_SYSTEM_FREETYPE=yes USE_SYSTEM_ZLIB=yes -j$(nproc)
```

MuPDF accepts `USE_SYSTEM_*` toggles to drop bundled thirdparty libs — see `mupdf/Makelists` and `mupdf/Makethird` for the full list (freetype, harfbuzz, zlib, jbig2dec, openjpeg, libjpeg, gumbo, mujs).

### Required system packages (Arch)

- `qt6-base qt6-svg qt6-declarative qt6-speech qt6-wayland` — Qt 6 + Wayland QPA plugin
- `harfbuzz sqlite zlib`
- `mupdf` (only if linking against system mupdf instead of the submodule)
- Tools: `cmake make pkg-config gcc`

`qt6-wayland` provides the `wayland` QPA plugin loaded at runtime; nothing in sioyek's build scripts has to be changed for Wayland.

### Forcing Wayland at runtime

Sioyek delegates platform selection to Qt. To force Wayland (and skip xcb entirely):

```bash
QT_QPA_PLATFORM=wayland ./sioyek    # one-shot
```

Or set it persistently in the desktop entry / shell env. If the desktop session is already Wayland and `qt6-wayland` is installed, Qt picks it automatically.

**Gotchas on Wayland that are worth knowing when editing the code:**

- `RunGuard` (`pdf_viewer/RunGuard.h/cpp`) uses `QLocalSocket` (Unix domain socket) for single-instance IPC — protocol-agnostic, works fine on Wayland.
- The portal/helper window position is set with `QWidget::move()` / `QWidget::setGeometry()`. **On Wayland, clients cannot position their own toplevel windows** — those calls become no-ops. If portal placement matters, the user has to position the window manually or rely on a compositor rule.
- Global screen geometry comes from `QGuiApplication::primaryScreen()->geometry()` in `main_widget.cpp` — works on Wayland.
- No keyboard/mouse grabs are used; pointer warping is not attempted.

### Optional Qt modules that can be dropped

The qmake build pulls these in unconditionally; the CMake build already drops
`TextToSpeech` behind `-DSIOYEK_NO_TTS=ON` (this checkout's default). For a
minimal build any of the others can be excised if the feature isn't wanted:

| Qt module | Source dependency | What you lose if removed |
|-----------|-------------------|--------------------------|
| `TextToSpeech` (`qt6-speech`) | `QtTextToSpeechHandler` in `utils.cpp` (guard at ~line 4294), `utils.h`, `main_widget.cpp` | TTS / read-aloud commands (**already dropped** here via `SIOYEK_NO_TTS`) |
| `QuickWidgets` + QML (`qt6-declarative`) | `pdf_viewer/touchui/*` and `pdf_viewer/qml/` | All touch-mode UI (mobile/tablet overlays) |
| `Svg` (`qt6-svg`) | Toolbar icons under `icons/*.svg`, `resources.qrc` | SVG toolbar icons |
| `Network` | Update check, paper download | Auto-update notification, paper downloads |

To remove a module: drop it from both `pdf_viewer_build_config.pro` (`QT += ...`) and `CMakeLists.txt` (`find_package(Qt6 REQUIRED COMPONENTS ...)` + `target_link_libraries`), then `#ifdef`-guard the call sites or excise them.

### Runtime assets and `resources.qrc`

A lot of what looks like a runtime dependency is actually **compiled into the binary** via `resources.qrc` (Qt resource system, accessed at runtime through `:/...` paths):

- `tutorial.pdf` — bundled
- `pdf_viewer/keys.config`, `pdf_viewer/prefs.config` — bundled defaults (loose-file copies override them)
- `pdf_viewer/shaders/*.fragment` and `*.vertex` — bundled
- All `pdf_viewer/touchui/*.qml` — bundled
- All `icons/*.svg` — bundled
- `data/command_docs.json`, `data/config_docs.json` — bundled, loaded via `QFile(":/data/...")` in `main_widget.cpp:load_command_docs()` (~line 10600)
- `data/embedding.npy`, `data/linear.npy`, `resources/fonts/JetBrainsMono.ttf` — **removed** from `resources.qrc` on this branch (commit `ce3e3f9e`). The npy files were dead code (`load_npy()` in `main.cpp` is commented out); the font is now referenced by name (`global_font_family = "JetBrains Mono"`, installed system-wide).

This means `shaders/` and `tutorial.pdf` copied next to the binary by `build_linux.sh` are mostly redundant — the resource fallback handles them. The loose `prefs.config` / `prefs_user.config` and `keys.config` / `keys_user.config` at runtime **are** still meaningful (they are the user-edit path).

### Config files

When `LINUX_STANDARD_PATHS` is defined (non-portable installs), config files live in `/etc/sioyek/` (defaults) + `~/.config/sioyek/` (user overrides) and data in `~/.local/share/sioyek/`. Define `NON_PORTABLE` (without `LINUX_STANDARD_PATHS`) to keep defaults beside the binary but put user files under XDG paths.

### Platform-specific preprocessor flags

- `SIOYEK_QT6` — set when Qt 6 is detected (always on this branch)
- `SIOYEK_ANDROID` — Android build (irrelevant here, but conditionally compiles out `synctex/`, `RunGuard`)
- `SIOYEK_MOBILE` — mobile UI layout
- `SIOYEK_DEVELOPER` — exposes developer-only commands in `input.cpp`
- `NON_PORTABLE` — use XDG user paths for config/data
- `LINUX_STANDARD_PATHS` — system install layout (`/etc/sioyek`, `/usr/share/sioyek`)
- `Q_OS_LINUX` / `Q_OS_MACOS` / `Q_OS_WIN` — Qt-provided OS detection

### Minimum Qt version

The source uses `#if QT_VERSION >= QT_VERSION_CHECK(6, 6, 0)` guards in a few places (`main_widget.cpp`). README says 6.7/6.8; that's the working minimum. This machine builds against Arch's `qt6-base` (currently 6.11.x). Upstream moved the build files to **C++20** (`CMAKE_CXX_STANDARD 20`) as of Sept 2026.

### Current submodule state of this checkout

- `mupdf/` submodule: rev `d189cc13` — checked out **and built** (`mupdf/build/release/libmupdf{,-third,-threads,-pkcs7}.a` present, built with the `znver4` flags). Rebuild only if the submodule pointer moves or the `.a` files go missing (see `UPDATING.md`).
- `zlib/` submodule: rev `21767c6` (checked out).

## Helper scripts (not part of the build)

`scripts/` contains user-facing Python helpers — `paper_downloader.py`, `embed_annotations_in_file.py`, `dual_panelify.py`, etc. — invoked by the user from the command palette, not linked into the binary. Ignore unless touching that feature.

## Architecture

### Core Class Hierarchy

**`MainWidget`** (`main_widget.h/cpp`) — top-level `QMainWindow` (or `QQuickWidget` in touch mode). Owns all other managers and handles high-level user interactions, command dispatch, and the portal window.

**`PdfViewOpenGLWidget`** (`pdf_view_opengl_widget.h/cpp`) — the OpenGL canvas. Renders PDF page textures, highlights, drawings, rulers, and overlays using custom GLSL shaders (`pdf_viewer/shaders/`).

**`Document`** (`document.h/cpp`) — wraps a MuPDF document (`fz_document`). Owns marks, bookmarks, highlights, portals, and freehand drawings for one PDF. Backed by `DatabaseManager`.

**`DocumentView`** (`document_view.h/cpp`) — viewport state (zoom level, scroll offset, two-page mode, ruler) over a `Document`. Multiple `DocumentView`s can reference the same `Document`.

**`PdfRenderer`** (`pdf_renderer.h/cpp`) — background thread pool that rasterizes pages via MuPDF and uploads textures to OpenGL. Requests are keyed by `RenderRequest` (path, page, zoom, slice).

**`DatabaseManager`** (`database.h/cpp`) — SQLite wrapper with two databases: a local per-directory DB and a global user DB. Persists opened book state, marks, bookmarks, highlights, portals.

**`ConfigManager`** (`config.h/cpp`) — parses `prefs.config` / `prefs_user.config` and exposes typed config values. Config types: `Int`, `Float`, `Color3/4`, `Bool`, `String`, `FilePath`, `FolderPath`, etc.

**`CommandManager` / `Command`** (`input.h/cpp`) — command pattern for all user actions. Each command subclass implements `next_requirement()` (to request further input like text/symbol/rect) and `perform()`. `InputHandler` translates key events into `Command` objects using the keybind config.

**`DocumentManager`** — cache of loaded `Document` objects, keyed by file path checksum.

### Coordinate System

Four distinct coordinate spaces defined in `coordinates.h`:
- `DocumentPos` — page number + (x, y) within that page (MuPDF page space)
- `AbsoluteDocumentPos` — concatenated y-axis across all pages (pages stacked vertically)
- `NormalizedWindowPos` — [-1, 1] OpenGL NDC
- `WindowPos` — pixel coordinates

Conversion methods are members of each struct (e.g., `DocumentPos::to_absolute(doc)`).

### Annotations / Persistence

`book.h` defines the annotation structs: `Mark`, `BookMark`, `Highlight`, `Portal`, `FreehandDrawing`. All inherit from `Annotation` (with UUID, creation/modification timestamps, JSON serialization). These are loaded from SQLite on document open and written back on close or edit.

### Touch UI

`pdf_viewer/touchui/` contains the mobile/touch overlay widgets (QML-driven). They are conditionally compiled and shown only in touch mode.

### Portals

A portal links a source location in one document to a destination in another (or the same) document. The portal window (`MainWidget` creates a second `PdfViewOpenGLWidget`) automatically tracks the nearest portal as the user scrolls.

### IPC

`RunGuard` ensures a single instance. If a second instance is launched with a file argument, it sends the path to the running instance via `QLocalSocket` and exits.

## Key Files

| File | Role |
|------|------|
| `pdf_viewer/main.cpp` | Entry point, argument parsing, manager instantiation |
| `pdf_viewer/main_widget.cpp` | Command dispatch, event handling, UI wiring |
| `pdf_viewer/input.cpp` | Command registry, keybind parsing, `InputHandler` |
| `pdf_viewer/document.cpp` | MuPDF document wrapping, text search, annotation I/O |
| `pdf_viewer/pdf_view_opengl_widget.cpp` | OpenGL rendering pipeline |
| `pdf_viewer/shaders/` | GLSL shaders for page rendering, dark mode, highlights |
| `pdf_viewer/prefs.config` | Default preferences (do not edit; use `prefs_user.config`) |
| `pdf_viewer/keys.config` | Default keybindings (do not edit; use `keys_user.config`) |
| `data/command_docs.json` | Documentation strings shown in the command palette |
| `data/config_docs.json` | Documentation strings for config options |
