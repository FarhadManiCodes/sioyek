# Personal patches & merge-conflict guide

What this `personal` branch changes on top of `upstream/development`, and where a
future `git merge upstream/development` (or, later, `nice_highlights`) is likely
to conflict. Regenerate the change list anytime with:

```bash
git log --oneline upstream/development..personal --no-merges
git diff --stat upstream/development...personal
```

## The patches

Grouped by concern. Exact commit list: `git log --oneline upstream/development..personal --no-merges`.

| Commit(s) | What | Touches | Conflict risk |
|-----------|------|---------|---------------|
| `9d9be81d` | `SQLite::SQLite3` → `SQLite3::SQLite3` target name | `CMakeLists.txt` | Low |
| `07d2b9be` | `SIOYEK_NO_TTS` opt-in build flag (disables Qt TextToSpeech) | `CMakeLists.txt`, `main_widget.cpp`, `utils.cpp`, `utils.h` | **High** |
| `e14066f9` | Don't force `LINUX_STANDARD_PATHS` (keep portable layout) | `CMakeLists.txt` | Medium |
| `fe4ee634` | Link bundled mupdf + harfbuzz + freetype on Linux | `CMakeLists.txt` | **High** |
| `ce3e3f9e`, `551f5ae5` | Strip dead `.npy` + bundled `JetBrainsMono.ttf` from `resources.qrc`; use system "JetBrains Mono" font by name; `-fvisibility=hidden` + `-Wl,-O2 -Wl,--as-needed` as `target_*_options` (`551f5ae5` made them target-scoped so they actually apply) | `CMakeLists.txt`, `pdf_viewer/main.cpp`, `resources.qrc` | **High** |
| `ec778073`, `0e08270a`, `a4e9ff3c`, … | Docs: `CLAUDE.md`, `UPDATING.md`, `PERSONAL_PATCHES.md` (+ ongoing edits) | new files | None |
| `9cdc6826` | gitignore `/build-cmake/` | `.gitignore` | None |

Real conflict surfaces: **`CMakeLists.txt`**, the three TTS source files
(`utils.h`, `utils.cpp`, `main_widget.cpp`), **`pdf_viewer/main.cpp`** (font
line, inside `main()` which upstream edits often), and **`resources.qrc`** (our
removals vs. upstream additions). `CLAUDE.md` / `UPDATING.md` /
`PERSONAL_PATCHES.md` / `.gitignore` are additive and never conflict.

## Conflict hotspots, file by file

### `CMakeLists.txt` — the most fragile (5 separate edits)

Upstream edits this file often, and the next-major `nice_highlights` branch adds
an RHI/Vulkan backend + high-quality TTS, which will touch the very lines below.

1. **Qt component list** — you replaced the inline `find_package(Qt6 ...)`
   component list with a `_sioyek_qt_components` variable so TTS can be dropped
   conditionally. If upstream adds/removes a Qt component, the merge will collide
   here.
   *Resolve:* keep your variable form; re-add any new upstream component to the
   `set(_sioyek_qt_components ...)` line.

2. **Link libraries** — you removed `mupdf` and `SQLite::SQLite3` from the
   portable `target_link_libraries`, moved TTS behind a conditional, and (Linux
   `else()` block) added explicit `-L mupdf/build/release -lmupdf -lmupdf-third
   -lmupdf-threads -lharfbuzz -lfreetype`. Upstream links system `mupdf`.
   *Resolve:* **keep your version** — this is the whole point of the bundled-mupdf
   build. Don't let an upstream "link system mupdf" change win.

3. **`LINUX_STANDARD_PATHS`** — you deleted the unconditional
   `target_compile_definitions(... LINUX_STANDARD_PATHS)` Linux block and left a
   comment. If upstream rewrites that block, you'll get a conflict.
   *Resolve:* keep it omitted (portable layout depends on this).

4. **`SQLite3::SQLite3`** — trivial rename; only conflicts if upstream also edits
   that line. Keep the `SQLite3::` form.

5. **Extra optimization flags** in the Linux `else()` block — see the dedicated
   section below (`CMakeLists.txt` extra flags — must stay target-scoped).

### `utils.h` — TTS class declarations  (**watch `nice_highlights`**)

You wrapped `QtTextToSpeechHandler` in `#ifndef SIOYEK_NO_TTS ... #else
NoOpTextToSpeechHandler ... #endif` and guarded the `<qtexttospeech.h>` include.
`nice_highlights` adds high-quality/cloud TTS handler classes right here.
*Resolve:* re-wrap any new upstream TTS handler in the same `#ifndef
SIOYEK_NO_TTS` guard, and add a matching no-op stub in the `#else` branch so the
factory keeps compiling.

### `utils.cpp` — TTS method bodies

You wrapped the `QtTextToSpeechHandler::` method definitions in `#ifndef
SIOYEK_NO_TTS ... #endif`. Same story as the header.
*Resolve:* extend the guard around any new TTS method bodies upstream adds.

> Note: the ligature `stable_sort` fix in `utils.cpp` is **upstream's**, not
> yours — don't mistake it for a personal patch.

### `pdf_viewer/main.cpp` — system font  (commit `ce3e3f9e`)

Upstream loads a bundled `:/resources/fonts/JetBrainsMono.ttf` via
`QFontDatabase::addApplicationFont(...)` at the top of `main()`. You replaced
that whole block with `global_font_family = "JetBrains Mono";` (the font is
installed system-wide). Upstream touches `main()` often, so expect the
surrounding lines to move.
*Resolve:* keep the one-liner; drop any re-added font-file loading. If upstream
switches to a *different* bundled font you may want the new name instead.

### `resources.qrc` — stripped assets  (commit `ce3e3f9e`)

You removed `data/embedding.npy`, `data/linear.npy` (dead code — `load_npy()` is
commented out in `main.cpp`) and `resources/fonts/JetBrainsMono.ttf`.
*Resolve:* keep them removed. If a merge re-adds them, delete again. If upstream
adds *new* `<file>` entries, keep those.

### `CMakeLists.txt` extra flags — must stay target-scoped  (commits `ce3e3f9e`, `551f5ae5`)

`-fvisibility=hidden -fvisibility-inlines-hidden` and `-Wl,-O2 -Wl,--as-needed`
live in the Linux `else()` branch as `target_compile_options(sioyek PRIVATE ...)`
/ `target_link_options(sioyek PRIVATE ...)`. They were originally written as
directory-level `add_compile_options` / `add_link_options`, which silently do
nothing because `qt_add_executable(sioyek)` is defined ~170 lines earlier —
`551f5ae5` fixed that.
*Resolve:* if a merge reverts them to `add_*_options`, switch back to the
`target_*_options(sioyek PRIVATE ...)` form.

### `main_widget.cpp` — TTS factory + include

Two small edits: guarded `<qtexttospeech.h>` include, and a
`#elif defined(SIOYEK_NO_TTS)` branch in `MainWidget::get_tts()` that returns
`NoOpTextToSpeechHandler`. If upstream restructures `get_tts()` (likely under
`nice_highlights`, which moves logic into controllers), reapply the `#elif`
branch wherever the factory lands.

## Strategy

- **Routine `development` merges:** low effort. Most upstream PRs don't touch
  these files; when they touch `CMakeLists.txt`, the resolution rule is almost
  always "keep mine" for the mupdf/paths/TTS lines, "take theirs" for unrelated
  additions.
- **The eventual `nice_highlights` merge:** expect real work on all four TTS
  touch points, because that branch reworks TTS and the build. Budget time, and
  reapply the `#ifndef SIOYEK_NO_TTS` pattern rather than trying to auto-merge.
- **If a TTS conflict ever gets messy**, the nuclear option is to set
  `-DSIOYEK_NO_TTS=OFF`, drop the source guards, install `qt6-speech`, and live
  with upstream's TTS — your build still works, just larger.

See `CLAUDE.md` for the full build rationale and `UPDATING.md` for the rebuild
runbook.
