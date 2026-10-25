# Third-party licenses

Bleed Engine's own code is MIT-licensed (see `LICENSE`). It vendors or ports
a few third-party libraries, each under its own permissive license, kept
alongside its source:

| Component | Where | License | Notes |
|---|---|---|---|
| cocos2d-x 2.1.1 (a subset) | `src/engine/cocos/` | MIT | `src/engine/cocos/LICENSE-cocos2d-x.txt`. Ported from cocos2d-x's source onto this engine's platform layer; class names kept as upstream. |
| lunasvg 3.5.0 | `third_party/lunasvg-3.5.0/` | MIT | `third_party/lunasvg-3.5.0/LICENSE`. Rasterises the engine's own splash art. |
| libtess2 | `third_party/libtess2/` | SGI Free Software License B, v2.0 | `third_party/libtess2/LICENSE.txt`. Permissive, OSI-style license; used to triangulate Flash vector fills for the GPU. |
| texture2ddecoder | `third_party/texture2ddecoder/` | MIT | `third_party/texture2ddecoder/LICENSE`. Decodes PVRTC-compressed textures on the CPU. |

Each vendored component's own license file is the authoritative text; this
table is a summary, not a substitute.
