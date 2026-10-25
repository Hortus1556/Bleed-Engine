# Bleed Engine

Bleed Engine is a native C++ game engine built to run Adult Swim /
Mediatonic's **Amateur Surgeon** series (the Flash games and their iOS
ports) on Windows, without a Flash player, without the original's online
services, and without any emulation layer — the original ActionScript is
decompiled, translated into data the engine runs directly, and its Flash
vector art is rendered on the GPU as actual triangles, not rasterised
frames.

**This repository is the engine only.** It contains no Amateur Surgeon
game content, art, audio, or decompiled script data, and none of the
per-game ports built on top of it. Those stay in a private workspace; this
repo is published so the engine itself — the parts that have nothing to do
with any one game — can be used, read, or built on independently.

## What's here

- **`src/core`, `src/platform`** — the main loop and its three threads
  (logic, display, texture upload), window/input/audio via SDL3, the save
  system, debug tooling.
- **`src/flash`** — a SWF reader, and a renderer that cuts every Flash
  shape, morph and static text into triangles once and draws them at any
  size with no texture (fills, strokes, gradients, bitmap fills), with a
  raster fallback for filtered art and dynamic text.
- **`src/surgeon`** — an ActionScript 1/2 runner: given a SWF's decompiled,
  indexed scripts (see `tools/index_scripts.py`), it plays a movie's
  timeline, buttons and frame scripts as data, with native "modules" for
  whatever a script can't express and is real game logic. This is meant to
  be linked into a game executable together with `BleedEngine`, not built
  as a standalone library.
- **`src/engine/cocos`** — a from-scratch port of the subset of cocos2d-x
  2.1.1 that a cocos-based iOS game needs on top of this engine's platform
  layer (nodes, actions, the scheduler, sprites, particles, grid effects).
- **`src/engine/ui`** — a subset of UIKit and Core Animation (views from
  compiled nibs, layer animations, basic controls) for games whose screens
  were originally UIKit.
- **`src/tools/extract_swf.cpp`** — reads a SWF and writes it out as the
  engine's package format (vector shapes as SVG, symbol timelines as JSON,
  audio, a script export for `tools/index_scripts.py`), with a byte-exact
  round-trip check.
- **`tools/`** — the Python side: `index_scripts.py` / `as2.py` parse
  decompiled ActionScript 1/2 into the data actions the runner executes,
  flagging what's native code or unported; `make_bleed_splash.py` writes
  the engine's own splash; `display_pace.py`, `motion_check.py`,
  `motion_sweep.py` check the display's timing and that only authored
  animation steps (never code-driven motion) move in discrete frames.
- **`third_party/`** — `lunasvg`, `libtess2`, `texture2ddecoder` (see
  `THIRD_PARTY_LICENSES.md`).

## What's *not* here, on purpose

Any game's content (`library.json`, vector art, audio, decompiled scripts),
any per-game C++ (`src/as1`, `src/as2`, `src/as3`, `src/christmas` in the
private workspace), and the original SWFs/IPAs. None of that is this
engine's to redistribute.

## Building

Requires CMake, a C++17 compiler, and SDL3 (`find_package(SDL3 CONFIG)`).

```
cmake -B build -G "Ninja"
cmake --build build
```

This builds the `BleedEngine` static library plus the `extract_swf` and
`svgrender` tools. There is no sample game in this repository yet — a game
links `BleedEngine`, compiles `src/surgeon` into its own executable, and
supplies its own package (see `src/surgeon/StageScreen.h` for what a game's
set-up needs to provide).

## License

Bleed Engine's own code is MIT (`LICENSE`). Vendored third-party code keeps
its own license; see `THIRD_PARTY_LICENSES.md`.
