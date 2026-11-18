# Modern Graphics for Dethrace

An investigation into how to bring modern graphics to [dethrace](https://github.com/dethrace-labs/dethrace) — and a working implementation of post-process shaders and FSR 1.0 upscaling in its BRender OpenGL driver.

---

## Background

**Dethrace** is an open-source, pure-C reimplementation of the 1997 game *Carmageddon*. It is built function-by-function through reverse engineering of the original executable: the game logic is ~72k lines of C under `src/DETHRACE/common`, built with CMake.

Rendering is not written against a modern graphics API directly. Instead dethrace drives **BRender**, the 1990s software renderer from Argonaut Software, with **SDL2** as the platform layer. The dethrace BRender fork already ships an OpenGL renderer (`glrend`) and a virtual-framebuffer device; the SDL2 harness presents that virtual framebuffer through an OpenGL 3.1 core / GLES 3.0 context.

That structure dictates what "modern graphics" can realistically mean for this codebase.

## Goal

Determine a practical path to modern graphics for dethrace — better output quality and access to modern techniques (upscaling, anti-aliasing, temporal effects) — **without rewriting the game**.

## Investigation summary

Two strategies were evaluated.

| Strategy | What it means | Verdict |
|---|---|---|
| **Engine replacement** (Unity / Unreal / Godot) | Port the game onto a complete engine with a retained scene graph, its own physics, and C#/C++ game scripting | **Rejected** |
| **Renderer enhancement** (chosen) | Keep the existing game logic and modernize BRender's device driver, which already has an OpenGL backend | **Chosen** |

### Why engine replacement was rejected

Unity, Unreal, and Godot are complete *game engines*, not renderers: each brings a retained scene graph, its own physics, its own object model, and C#/C++ scripting. Dethrace's logic is tightly fused with BRender's immediate-mode rendering:

- **72%** of source files reference BRender types.
- Physics and AI mutate `br_actor` transforms **directly**.

Adapting the game to an engine's scene graph would therefore touch essentially all game code — a full rewrite of the game, not an upgrade, with a very high lift and no incremental delivery path. Rejected.

### Why renderer enhancement was chosen

The rendering abstraction boundary already exists. The BRender fork ships an OpenGL renderer (`glrend`) and a virtual-framebuffer device, and the SDL2 harness presents that framebuffer through a GL 3.1 core / GLES 3.0 context. Modern rendering techniques can be layered into the device driver while the game logic and BRender API stay untouched — an **incremental**, testable path.

## What's been done

### Option A — post-process shaders (proof of concept)

Option A validates the lowest-risk end of the enhancement path: intercept the frame at present time and apply GPU shaders.

| Step | Work | Result |
|---|---|---|
| **Baseline** | Build the unmodified tree green and verify it renders on the Splat Pack full game and the freeware demos | Green build; runtime-verified on the OpenGL path |
| **Increment 1** | Re-enabled off-screen FBO render-to-texture: render the 3D scene into a color + depth target, then blit to the window | Behavior-preserving — output byte-identical to baseline |
| **Increment 2** | Added a sharpen post-process: a 4-neighbor unsharp mask applied only to the present blit | First visible effect; amount runtime-tunable via `GLREND_SHARPEN_AMOUNT` (default `0.4`) |

**Increment 1 detail.** The scene renders into a `GL_RGB8` color target plus a `GL_DEPTH_COMPONENT24` depth buffer, then blits to the window. The target is lazily (re)sized as the window changes.

**Increment 2 detail.** The unsharp mask reads the FBO color texture and writes to the default framebuffer in the present blit. The overlay path is untouched.

**Measured results.** Sharpness was measured as in-game Laplacian variance against the baseline frame:

| `GLREND_SHARPEN_AMOUNT` | Laplacian-variance sharpness | Color drift |
|---|---|---|
| `0.4` (default) | **+40%** | Negligible |
| `0.8` | **+80%** | Small |

The effect is visually subtle despite the numbers: the source art is 8-bit paletted 640×480, which is the hard ceiling on what any post-process can recover. As a shipping feature, sharpen alone is not compelling. As a **proof of concept**, it establishes two things:

1. The full loop works — modify renderer → build → run → measure.
2. The reusable **FBO + present-blit hook** works, and every future shader or upscaler hangs off it.

### Option B — FSR 1.0 spatial upscaling (implemented and shipped)

Option B is the first shippable feature built on that hook: **AMD FidelityFX FSR 1.0** (EASU spatial reconstruction + RCAS sharpening), vendored from the MIT-licensed AMD reference implementation.

- The scene now renders at its **native 640×480** into the off-screen FBO, instead of at drawable size, so the upscaler always receives a clean, consistently sized source image.
- **EASU** reconstructs the frame up to the **letterboxed window viewport** (e.g. a 1440×1080 viewport inside a 1920×1080 window).
- **RCAS** sharpens at output resolution and replaces the legacy unsharp mask while FSR 1.0 is active.
- FSR 1.0 is **on by default for the OpenGL core path** and **off by default for OpenGL ES**; it is only used when the window is larger than 640×480.
- Verified in game: a noticeably better image than the old 2.25× nearest upscale — smooth reconstructed edges instead of blocky pixels.
- Runtime toggles are wired through command-line flags, `dethrace.ini`, and the renderer's environment settings (see [Usage / toggles](#usage--toggles)).

## Usage / toggles

FSR 1.0 and the sharpen post-process can be controlled from three places. All three write the same settings, and dethrace forwards the resolved values to the renderer before it initialises.

Order of precedence (highest first):

1. Command-line flag
2. `[Graphics]` entry in `dethrace.ini`
3. Existing `GLREND_*` environment variable
4. Renderer default

Only settings that are specified at a level are applied: an existing `GLREND_*` environment variable is left untouched when neither the command line nor `dethrace.ini` sets that setting, so the renderer defaults still apply.

### Command-line flags

| Flag | Values | Description |
|---|---|---|
| `--fsr` / `--no-fsr` | | Force the FSR 1.0 upscale on/off (same as `GLREND_FSR=1`/`0`). |
| `--fsr-sharpness=<n>` | `0`..`2` (stops) | RCAS sharpening strength while FSR 1.0 is active: `0` = maximum sharpening, higher values are softer. Also accepted as `--fsr-sharpness <n>`. |
| `--sharpen=<n>` | `0`..`~1` | Strength of the legacy present-path unsharp mask, used when FSR 1.0 is not active. Also accepted as `--sharpen <n>`. |

### `[Graphics]` section of `dethrace.ini`

| Key | Values | Description |
|---|---|---|
| `FSR` | `0`, `1` | FSR 1.0 upscale on/off (same as `GLREND_FSR`). |
| `FSRSharpness` | `0`..`2` (stops) | RCAS sharpening strength while FSR 1.0 is active (same as `GLREND_FSR_SHARPNESS`; default `0.2`). |
| `SharpenAmount` | `0`..`~1` | Strength of the legacy present-path unsharp mask, used when FSR 1.0 is not active (same as `GLREND_SHARPEN_AMOUNT`; default `0.4`). |

Key names are case-insensitive, and `fsr_sharpness` / `sharpen_amount` are accepted as aliases.

Example:

```ini
[Graphics]
FSR = 1
FSRSharpness = 0.2
SharpenAmount = 0.4
```

```sh
dethrace.exe --opengl --fsr --fsr-sharpness=0.2
dethrace.exe --opengl --no-fsr --sharpen=0.4
```

Values must be non-negative. An invalid command-line value aborts startup with an error; an invalid `dethrace.ini` value is ignored with a warning in the log. These settings only affect the OpenGL renderer (`--opengl` / `Emulate3DFX`).

## Architecture

The change is a presentation-stage hook, not a new rendering path:

```text
Game logic  (src/DETHRACE/common)
    │  BRender immediate-mode calls (br_actor, br_model, ...)
    ▼
BRender device driver: glrend
    │
    ├─ 3D scene + 2D overlay composited into an off-screen FBO
    │     color: GL_RGB8          depth: GL_DEPTH_COMPONENT24
    │     native 640×480, independent of the window size
    │
    └─ present pass into the letterboxed window viewport
         FSR 1.0 (default on GL core):
             EASU  640×480 ──▶ viewport   RCAS at output resolution
         legacy path (GLES / --no-fsr):
             nearest upscale + optional unsharp mask
              (OpenGL 3.1 core / GLES 3.0 context)
```

**Key pieces** (all under `lib/BRender-v1.3.2/drivers/glrend/`)

| Piece | Role |
|---|---|
| `DevicePixelmapGLEnsureRenderbuffers()` | Lazily allocates and (re)sizes the off-screen color + depth targets at native 640×480 |
| Present path / `RenderFullScreenTextureToFrameBuffer()` | Presents the finished frame into the letterboxed viewport; dispatches between the FSR 1.0 and legacy paths |
| `post.vert.glsl` / `post.frag.glsl`, `shader_post.c` | Legacy present blit — nearest upscale plus optional unsharp mask |
| `fsr1_easu.frag.glsl`, `fsr1_rcas.frag.glsl`, `shader_fsr1.c` | EASU reconstruction and RCAS sharpening programs, including shader loading and uniform plumbing |
| `fsr1_constants.c/.h`, vendored `fsr1/ffx_fsr1.h` | CPU-side FSR constants computed with the MIT-licensed AMD reference implementation |
| `glrend_settings.c/.h` | Reads the `GLREND_*` environment variables and applies renderer defaults |

**Code state.** All changes live under `lib/BRender-v1.3.2/drivers/glrend/` in the BRender submodule: the off-screen FBO present hook and post-process shaders (Option A), plus the vendored FSR 1.0 implementation, native-resolution rendering, and the EASU/RCAS present pass (Option B). The work is on branch `feature/gl-postprocess` of the project fork; it is **not** merged or upstreamed.

## Build & run

### Prerequisites

| Requirement | Version / notes |
|---|---|
| CMake | **≥ 3.20** |
| C compiler | MSVC, GCC, or Clang |
| SDL2 development library | Tested with **2.32.10** |
| Carmageddon game data | **Not included** — you must supply your own (see [Licensing](#licensing)) |

### Configure and build

Point `CMAKE_PREFIX_PATH` at the SDL2 development folder (the directory containing `include/` and `lib/`). Use whatever generator and configuration you prefer; the example below is Visual Studio 2022 (x64).

```sh
cmake -S . -B build -G "Visual Studio 17 2022" -A x64 \
      -DCMAKE_PREFIX_PATH=<path-to-SDL2-dev>
cmake --build build --config Debug
```

For a single-config generator such as Ninja or Makefiles:

```sh
cmake -S . -B build -G "Ninja" -DCMAKE_PREFIX_PATH=<path-to-SDL2-dev>
cmake --build build
```

### Run

1. Copy `dethrace.exe` and the SDL2 runtime DLL (`SDL2.dll`) into your Carmageddon game folder.
2. Launch it with the OpenGL renderer and cutscenes disabled:

   ```sh
   dethrace.exe --opengl -nocutscenes
   ```

3. OpenGL mode requires the game's **3dfx assets** — specifically `DATA\PIXELMAP\SMOKE.PIX`. Freeware demos that lack these assets cannot use the OpenGL path.
4. FSR 1.0 is active by default on the OpenGL core path. Toggle it with `--no-fsr` / `--fsr`, or adjust sharpness as described in [Usage / toggles](#usage--toggles).

## Roadmap

| Option | Technique | Inputs required | Effort | Status |
|---|---|---|---|---|
| **A** | Post-process sharpen on the present blit; FBO + present hook | Color texture | Small | **Implemented** on `feature/gl-postprocess` |
| **B** | FSR1 spatial upscale (AMD FidelityFX FSR 1.0 — EASU + RCAS) | Color texture only | ~1 week on top of A | **Implemented** — shipped on `feature/gl-postprocess` |
| **C** | Temporal upscaling (FSR 2 / FSR 3 or DLSS) | Depth texture, per-pixel motion vectors, color history, jitter | ~2–4 months | **Planned** |
| **A-variants** | FXAA, LUT-palette, CRT shaders | Color texture only | Days each | Optional — cheap drop-ins on the same hook |

**Option B — FSR1 spatial upscaling.** Color-only, single-pass EASU + RCAS, plugged into the same present blit. Implemented and verified in game.

**Option C — temporal upscaling.** The hardest part is not the upscaler but the inputs: the pipeline currently produces no depth texture, no per-pixel motion vectors, and no color history, and it does not jitter the projection. Camera motion is easy to derive; *moving actors* require a renderer-state change plus game-side plumbing to emit per-object motion. That is why this is a multi-month effort rather than a shader drop-in.

**A-variants.** FXAA, palette LUT, and CRT shaders are cheap because they need only the color texture and the present-blit hook already in place.

## Licensing

| Component | License |
|---|---|
| dethrace | **GPL-3.0** (the upstream README additionally notes "non-commercial") |
| BRender | **MIT** |
| FSR1 (AMD FidelityFX) | **MIT** |
| Carmageddon game assets | **Copyrighted** — THQ Nordic / Stainless. **Not included.** |

This project does not distribute game data. To run it you must supply your own legally obtained Carmageddon assets.

## Status

- **Done:** Option A (off-screen FBO, behavior-preserving present hook, sharpen post-process PoC) and Option B (FSR 1.0 EASU + RCAS upscaling at native 640×480) — both implemented and pushed on `feature/gl-postprocess`. Runtime toggles are wired through command-line flags, `dethrace.ini` `[Graphics]`, and `GLREND_*` environment variables.
- **Remaining:** Option C (temporal upscaling) is planned; FXAA / LUT-palette / CRT are optional cheap additions on the existing hook. The branch is a fork and is not upstreamed.
