# Updating / Rebuilding this sioyek checkout

Personal `znver4` Arch/Wayland build. Full build rationale lives in `CLAUDE.md`;
this file is just the recurring **pull → rebuild → install** runbook.

## TL;DR (routine update)

```bash
cd ~/Installs/sioyek
git fetch upstream
git merge upstream/development          # onto the 'personal' branch
cmake --build build-cmake -j$(nproc)    # incremental; reconfigures itself if CMakeLists changed
cp build-cmake/sioyek ~/.local/share/sioyek/sioyek
```

The wrapper `~/.local/bin/sioyek` exec's `~/.local/share/sioyek/sioyek`, so the
`cp` *is* the install. No `make install`, no sudo.

## When do I actually need to recompile?

Check what changed since the installed binary:

```bash
stat -c '%y' ~/.local/share/sioyek/sioyek          # install time
git log --oneline --after="<that date>" personal    # commits since
git diff --stat <last-built-commit> HEAD -- pdf_viewer/ '*.cpp' '*.h'
```

- Touches `pdf_viewer/**.cpp`/`**.h`, shaders, or `CMakeLists.txt` → **rebuild**.
- Only docs / `.gitignore` / completions / qmake `.pro` → **no rebuild needed**.

## When do I need to rebuild MuPDF too? (the slow part)

Usually **not**. MuPDF is a bundled submodule, built once. Rebuild it only if:

- the `mupdf/` submodule pointer moved (`git submodule status` shows a new rev), or
- `mupdf/build/release/libmupdf.a` is missing.

```bash
cd mupdf
make XCFLAGS="-march=znver4 -O3 -flto=auto" USE_SYSTEM_HARFBUZZ=yes \
     USE_SYSTEM_FREETYPE=yes USE_SYSTEM_ZLIB=yes -j$(nproc)
cd ..
```

## First-time / from-scratch configure

Only needed if `build-cmake/` doesn't exist or you want to change build options.
The persisted cache already has these; a fresh configure must re-pass them:

```bash
cmake -B build-cmake -DCMAKE_BUILD_TYPE=Release \
      -DSIOYEK_NO_TTS=ON \
      -DCMAKE_CXX_FLAGS="-march=znver4 -O3 -flto=auto -pipe -fno-plt"
```

- `SIOYEK_NO_TTS=ON` → drops Qt TextToSpeech; no `qt6-speech` package needed.
  Set `OFF` (and `pacman -S qt6-speech`) if you ever want read-aloud back.
- MuPDF (step above) must already be built before this works.

## Known-benign configure output (ignore)

- `Could NOT find WrapVulkanHeaders` — Qt's optional Vulkan probe; sioyek renders
  via OpenGL, never used.
- (Fixed) SQLite deprecation warning — CMakeLists now uses `SQLite3::SQLite3`.

## Sanity checks after install

```bash
ldd ~/.local/share/sioyek/sioyek | grep -i speech   # expect nothing (TTS off)
~/.local/bin/sioyek --help                          # launches cleanly
```

## Branch notes

- `personal` = your branch; `upstream/development` = where upstream PRs land.
- `upstream/nice_highlights` is the next-major work-in-progress (RHI backend,
  controller refactor, high-quality TTS). Pre-release; pull only after it merges
  into `development`.
