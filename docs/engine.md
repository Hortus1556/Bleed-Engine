# Bleed Engine

Bleed Engine (the `BleedEngine` library: `src/core`, `platform`, `engine`,
`flash`, and the shared "surgeon" game runner in `src/surgeon`) is a
general engine for running a Flash (and, via a mirrored-class layer not
included in this repository, an iOS) game natively, with its original
ActionScript translated into data the engine runs directly rather than
emulated.

This repository is the engine only — no game's content or per-game code is
included (see the root `README.md`). This page describes what exists:
- the engine's **splash**, shown before a game starts;
- the **loop and the display**: logic passes on one thread, the display on
  its own, textures uploaded on a third; art in shared pages, drawn in
  batches;
- the **game package**, the folder a game's content lives in;
- the **script index**, what each of a Flash game's scripts does;
- the **stage**, which runs a game by its own translated scripts, with
  native modules where the scripts are real logic;
- the **UIKit layer**, for games whose screens come from iOS nibs;
- the **cocos layer**, cocos2d-x 2.1.1's scene graph for games built on it.

## The splash: "Powered by Bleed Engine"

A game can open on the engine's own splash before its first screen
(`platform/Splash.cpp`, called by the shared main loop, `runGame` in
`platform/GameMain.h`): the wordmark, "POWERED BY" over a blood-red "BLEED
ENGINE" with drips running from its foot, fades in on black (0.6 s), holds
(1.8 s) and fades out (0.6 s); a click or Esc goes straight to fading out.
The wordmark is vector art in the engine's own angular stroke lettering (no
font file): `tools/make_bleed_splash.py` writes it as an SVG string into
`platform/BleedSplashArt.h` (rerun the tool to change it; `--svg <file>`
also writes a preview), and the splash rasterises it with lunasvg at the
window's resolution, again whenever that changes, at 70% of the game
screen's width (at most half its height). It is drawn through the recorded
drawing like everything else but is not one of the game's frames, and
while debug automation drives a run it is left out, so tests and their
frame numbers start at the game's own first screen.

## The loop and the display: three threads

A game runs in logic passes of a fixed length (`GameInfo::passSeconds`),
on the main loop (`runGame`, `platform/GameMain.h`). Each pass updates the
game and records one frame of draws (`platform/DrawList`); nothing is
drawn to the window while it runs. Three threads share the work, each with
its own GL context, the textures shared between them:
- **The game's thread** (the main one): the window's events, the logic,
  the recording, the vector art's triangles (`flash::MeshCache`, below)
  and what art is still rasterised (`flash::Raster`, its workers making
  it). It makes no GL call while it records and touches no texture's
  texels itself.
- **The display thread** (`presentLoop`, `Platform.cpp`): shows the
  recorded frames, once per display refresh (vsync; at most 240 a second
  without it), at the window's native resolution. Each frame is shown on
  its own clock: the frame a pass records is due a short lead after its
  pass was (`platform::setFrameDue`), so the display never waits on a
  pass, and what the player does shows a few milliseconds after the pass
  that saw it. Between its due time and the next frame's, it is eased in
  from the frame before (smooth motion, below). A frame not ready by then
  holds the one before at its end; nothing else stops the display.
- **The texture thread** (`textureWorkLoop`, `DrawList.cpp`): the texels of
  every texture the game's thread makes (`platform::setTexels`,
  `setTexelsAt`: the raster cache's art) are uploaded here as soon as
  they're queued, and textures let go are deleted here, in order. A frame
  is shown only once all its textures are in. It also makes textures ahead,
  named and made in its context, which `platform::newTexture` hands out,
  so the game's thread names a texture with no GL call. The display thread
  and the texture thread take turns (`DrawTurn`): uploads go in strips
  between the display's draws.

**No GL call while recording.** Game code's matrix calls (`glMatrixMode`,
`glOrtho`, `glTranslatef`, `glPushMatrix`, `glGetFloatv` of a matrix...) go
to a matrix stack the engine keeps on the CPU (`platform/GLMatrices.cpp`,
the names redirected in `GL.h`), which recording reads; the replay sets
GL's own.

**Vector art on the GPU.** A Flash game's art is vectors, and the GPU draws
it as the Flash player did: each shape, morph step and static text is cut
into triangles once (`flash/VectorMesh`: fills' outlines by libtess2,
even-odd, curves flattened to within 0.2 pixels at the size they're drawn;
strokes from their segments, joins and caps), cached per size step and
morph step, and drawn at any size, place, turn and colour transform with
no texture. Solid colours are per vertex; gradients are small textures and
bitmap fills their image, the triangles mapping them by the fill's matrix.
Edges are smoothed by multisampling. The replay transforms the vertices on
the CPU into the same batches as the quads (`drawVector`, `drawVectorAt`),
indexed, skipping meshes off the game screen; smooth motion moves a mesh by
its place exactly as a quad of art. What's still rasterised to textures
(`flash::Raster`): pieces under filters (blur, glow, shadows), text fields,
a gradient under an added colour.

**Shared pages and batched draws.** Art up to 256 texels a side is packed
in shared texture pages (`platform/TextureAtlas`); bigger art goes in
larger pages cut into shelves. New art goes in the fullest page that holds
it, a slot is let go as a texture is, and a page with nothing left goes.
The replay then sends consecutive quads with the same texture and blending
as one draw (`flushBatch`).

**On Android** an EGL surface is current in one context at a time: the
game's and the texture thread's contexts are current with no surface
(`EGL_KHR_surfaceless_context`), and only the display thread's has the
window's. When the app goes to the background, the display thread lets go
of the surface and waits; back, it takes the new surface.

**Checking the display's pace.** With `AS_DISPLAY_LOG=<file>` (debug mode)
the display thread logs each display frame; `tools/display_pace.py <file>`
reads it and reports holds (the display standing still), jumps (catching
up) and slow display frames. `tools/motion_check.py` / `motion_sweep.py`
check, in numbers rather than only screenshots, that code-driven motion is
smoothed between passes while authored animation (a swapped pose, a frame
of a frame-by-frame animation) steps exactly at its pass.

## The package: a game's `assets` folder

One folder per game holds its content — the engine's "package." This
repository doesn't ship one; the layout a game's own package follows:

| Path | Holds |
|---|---|
| `library.json` | stage size, frame rate, background, Flash version; linkage names (`exports`); the ids of every shape, morph, sprite and font |
| `vector/shapes/<id>.svg` | each shape, as SVG (dialect below) |
| `vector/morphs/<id>_start.svg`, `<id>_end.svg` | each morph shape's two ends, same path order |
| `vector/symbols/<id>.json` | each sprite's timeline: frame count, labels, per-frame place/remove ops, scripts, plain frame actions |
| `vector/buttons.json` | each button's records: states, character, depth, matrix, colour transform |
| `vector/texts.json`, `vector/edittexts.json` | static texts and edit fields |
| `vector/fonts/<id>.json` | name, style, metrics, character codes and advances; each glyph an SVG |
| `audio/<name>.mp3` / `.wav`, `audio/index.json` | each sound, by its linkage name |
| `scripts/` | the decompiled ActionScript (JPEXS FFDec's export) and `index.json` (below) |
| `data/` | the game's own data (its own choices: dialogue, generated gameplay data, module assignments) |

### The SVG dialect (`flash/SvgShape.*`)

Each file is valid SVG: a browser or an editor shows the shape. It's also
a fixed dialect that reads back to exactly the same shape.

| Element | Content |
|---|---|
| `<svg>` | `viewBox` in pixels; `data-bounds` is the shape's bounds in twips |
| `<defs>` | one gradient per gradient style, in Flash's gradient-square coordinate space |
| `<g data-layer="n">` | one per style layer |
| `<path data-fill="i">` | even-odd fill paths in drawing order |
| `<path data-line="i">` | stroke paths in drawing order, with caps, join, miter limit and width |

Coordinates are pixels with two decimals, which is exactly Flash's twips
(1/20 px). Paths use `M`, `L` and `Q` only.

## Extracting: `extract_swf`

`extract_swf <movie.swf> <package dir>` (`src/tools/extract_swf.cpp`):
1. Parses the SWF (`flash/Swf.*`) and writes the package (`flash/Package.*`),
   replacing `library.json`, `vector/` and `audio/`. It leaves `data/` and
   `scripts/` alone.
2. Checks the round trip: reads the package back, writes it again beside
   it, and requires every file to be byte-identical.

## Loading

`VectorArt::library("library.json")` reads a package through `assets::read`
into the same in-memory model the SWF parser fills. Shapes are read when
first drawn and let go after being unused a while; most of a game's art is
not on screen at any one time. `VectorArt::preload(name)` starts reading a
package on a worker thread, for `library(name)` to take when it's first
asked for.

**Pictures.** `platform::decodeImage` turns a picture file into
straight-alpha RGBA. PNG/JPEG go through stb_image; PVR textures (PVRTC or
uncompressed) are decoded on the CPU by `third_party/texture2ddecoder`,
since desktop GL has no PVRTC; a CCZ file is inflated first.

**Sounds** are decoded whole when loaded (`engine/MediaPlayer`): Apple's
CAF natively, MP3/WAV through miniaudio, AAC through the platform.

## The script index

`tools/index_scripts.py <game> [--swf <movie.swf>] [--report] [--diff
<other game>]` reads every script of a package's `scripts/` (exported with
FFDec from `--swf` when missing), parses it into a syntax tree
(`tools/as2.py`: ActionScript 1/2 as FFDec writes it), and sorts each
statement:

| Kind | What | Kept as |
|---|---|---|
| **translated** | statements the runner can do: timeline control, variable/property sets, `if`/loops/`return`, calls, `new` of Flash's built-ins, functions and handlers whose bodies are all of this | actions in `scripts/index.json` |
| **native** | what the runner can't do (`switch`, `with`, `try`, a computed class...), or what a module claims | listed with what it is, and the module that implements it |
| **unported** | native with no module | the report's checklist |

**Modules: `data/modules.json`** maps native code to the C++ that
implements it, matched by regex over a statement's source, by recurring
pattern, by function/button/frame/sprite name — the first match wins.

**`--report`** prints the counts, the `_root` variables used, the screens,
the functions called, the sounds named, and the native statements per
module (and every unported one) — the checklist for a new port.

## Running the scripts

`src/surgeon/Script.*` runs the index's translated actions:
- **Values.** `script::Value` holds ActionScript 2 values: undefined,
  null, booleans, numbers, strings, objects, Arrays, movie clips.
  Conversions and loose `==` follow AS2; comparisons follow AVM1 (SWF 7+):
  `undefined` is NaN in arithmetic, and `a <= b` is `!(a > b)`.
- **Clips.** `script::Clip` is any clip the scripts can reach. A reference
  to a removed clip reads as undefined, as AVM1 resolved one.
- **Runner.** `script::Runner` evaluates expressions and runs actions.
  `runFrame` runs a clip's frame script, `runButton` a button's handler,
  `runEnterFrames` the `onEnterFrame` handlers scripts set.
- **Calls** reach, in order: the clip's own methods, a native function a
  module registered, then the language's built-ins.

`src/surgeon/Scene.*` puts the runner on a tree of `ArtNode`s — a node's
own clip, or an inner clip of its symbol promoted to a node once a script
needs to address it directly (`attachMovie`, `swapDepths`, a transform
property set) — walking and running frame scripts each pass in Flash's own
order.

## The stage: a game run by its own scripts

`src/surgeon/StageScreen.*` is a game's screen: the movie's main timeline
as an `ArtNode`, run by a `Scene` and `Runner` over the package's index,
with Flash's built-in classes (`Sound`, `XML`, `SharedObject`, `Mouse`/`Key`
listeners), buttons run as Flash ran them, text fields showing their
variables, and **modules** — native code for a sprite on the main
timeline, which either replaces the clip (drawn in its place, taking the
pointer, given its own clip name for the scripts) or adds to it (drawn
over it, with its own buttons and keys).

A module that is real C++ shares the game's state with the scripts through
`Scene::bind`: a `_root` variable is a field of the game's code.

A game built on the stage is then: a package and its index; a
`StageScreen` with its save format, transition variable and modules; the
modules its index's report asks for.

## cocos2d-x: the cocos layer

`src/engine/cocos/` is the subset of cocos2d-x 2.1.1 that a cocos-based
iOS game needs, ported from its MIT source (the notice is in
`src/engine/cocos/LICENSE-cocos2d-x.txt`).
- **Names kept.** The classes keep cocos2d's names in namespace `cocos2d`
  (`CCNode`, `CCSprite`, `CCDirector`...), so a game mirrored from its
  binary reads against the decompile.
- **No game in it.** It's general: a feature is added to it as cocos2d
  2.1.1 had it.
- **Logic as 2.1.1 had it:** transforms, z order and order of arrival,
  onEnter/onExit/cleanup, the scheduler's timers and per-frame updates by
  priority, actions and ease curves, touch dispatch, scene changes, sprite
  frames from TexturePacker sheets.
- **Left out:** the director's display link, its own renderer and GL state
  cache, shaders, scripting, the camera, the platform layers — that work
  goes to this engine instead.
- **On the engine:** the game calls `CCDirector::mainLoop()` once a pass,
  with the pass length as its delta time, so play is deterministic and the
  engine smooths what moves between passes; drawing goes through the
  recorded drawing; `CCFileUtils` finds files in the package.

## UIKit: views from nibs

`src/engine/ui/` is the subset of UIKit and Core Animation that a game
whose screens are UIKit views needs. It's general: no game's names are in
it.
- **Nibs.** A compiled nib (Apple's NIBArchive format) decodes to JSON (the
  view tree, the File's Owner's outlets and actions); `ui::loadNib` builds
  the views from it.
- **Views.** `View`, `ImageView`, `Button`, `TextView`, `GLView` (a hook
  draws a game's own GL scene into it), `AlertView`, `Window` (touches,
  timers, animations, drawing, the responder chain).
- **Core Animation.** Layer animations by key (basic, keyframe) and the
  window's `CATransition`.
- **Time.** Everything runs on the pass clock, so what an animation moves
  is smoothed between passes like everything else the engine draws.
