# Repository Guidelines

## Project Structure & Module Organization

Sioyek is a Qt 6/C++20 PDF viewer. Core application code lives in `pdf_viewer/`; `main_widget.*` coordinates the UI, `document*` models PDFs, `pdf_renderer.*` renders through MuPDF, and `touchui/`, `qml/`, and `shaders/` contain presentation code. Icons and packaged metadata live in `icons/` and `resources/`, while `resources.qrc` lists assets embedded in the executable. `mupdf/`, `zlib/`, and `fzf/` are bundled dependencies; avoid incidental edits there. Python integrations are under `scripts/`, Android-specific files under `android/`, and release automation under `.github/workflows/`.

This checkout carries local build customizations. Read `PERSONAL_PATCHES.md` before changing CMake, TTS, fonts, or resources.

## Branch & Update Workflow

Work only on the `personal` branch. Synchronize exclusively with `origin/personal`; do not fetch, inspect, compare, merge, or rebase any `development` branch or the `upstream` remote unless the user explicitly requests it. Before making changes, update from `origin/personal`. Keep local commits linear on top of the remote branch (rebase local-only commits when necessary) and avoid duplicate merge commits. Build, install, and commit completed work on `personal`.

## Build, Test, and Development Commands

- `git submodule update --init --recursive` initializes bundled dependencies.
- `cd mupdf && make USE_SYSTEM_HARFBUZZ=yes -j$(nproc)` builds the MuPDF libraries required by Sioyek.
- `cmake -S . -B build-cmake -G Ninja -DCMAKE_BUILD_TYPE=Release -DSIOYEK_NO_TTS=ON -DCMAKE_CXX_FLAGS="-march=znver4 -O3 -flto=auto -pipe -fno-plt"` configures this machine's preferred build.
- `cmake --build build-cmake -j$(nproc)` invokes Ninja for an incremental build; run `./build-cmake/sioyek` for a smoke test.
- `./build_linux.sh` exercises the upstream qmake-based portable Linux build and writes output to `build/`.

Qt 6.7 or 6.8+, HarfBuzz, SQLite, and zlib development packages are expected. Do not commit generated `build/`, `build-cmake/`, or MuPDF build artifacts.

## Coding Style & Naming Conventions

Match nearby C++ rather than reformatting unrelated code. Use four spaces, braces on the same line, `snake_case` for functions and variables, and `PascalCase` for classes and structs. Keep matching declarations and definitions in `.h`/`.cpp` pairs. No repository-wide formatter is configured; keep diffs focused and preserve existing Qt and standard-library include conventions.

## Testing Guidelines

There is no dedicated automated unit-test suite. Every change should compile with CMake and receive a focused manual check using a representative PDF. For UI changes, exercise the affected command, keyboard/mouse path, and Wayland behavior. For build-system edits, also inspect `build-cmake/compile_commands.json` and test the relevant `SIOYEK_NO_TTS` configuration.

## Commit & Pull Request Guidelines

Recent history favors short, imperative subjects such as `Fix a bug where ...`; scoped maintenance subjects like `build: ...` and `docs: ...` are also used. Keep each commit single-purpose and explain issue links in the body. Pull requests should describe behavior and platform tested, list build/manual verification, link relevant issues, and include screenshots or a short recording for visible UI changes.
