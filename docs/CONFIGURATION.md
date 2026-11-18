# Configuration

Dethrace looks for a `dethrace.ini` file in the [system preferences directory](https://wiki.libsdl.org/SDL2/SDL_GetPrefPath):
| Platform | Example Path                                                                 |
|----------|----------------------------------------------------------------------|
| Windows  | `C:\Users\<NAME>\AppData\Roaming\dethrace\dethrace.ini`                |
| macOS    | `/Users/<NAME>/Library/Application Support/dethrace/dethrace.ini`      |
| Linux    | `/home/<NAME>/.local/share/dethrace/dethrace.ini`                      |


If this file is not present, dethrace will run with default configuration and attempt to discover the correct Carmageddon directory to use (see below).

## Example dethrace.ini file
```ini
[General]
; Enable original CD check
CdCheck = 0

; Enable original censorship check
GoreCheck = 0

; set to 0 to disable
FPSLimit = 60

; Full screen or window
Windowed = 1

; 3dfx mode (via OpenGL)
Emulate3DFX = 0

; Censored zombie/robots mode
BoringMode = 0

; Play cut scenes on startup and between races
Cutscenes = 0

; "hires" mode is 640x480, otherwise default 320x200
Hires = 1

; Only used in 'demo' mode. Default demo time out is 240s (4 mins)
DemoTimeout = 240

; Which directory in the [Games] section to run
DefaultGame = c1

; Original physics updates every 40ms, causing noticable camera juddering when cornering at slow speed
; Enabling this runs physics every frame, smoothing movement out
PhysicsPerFrame = 1

[Graphics]
; FSR 1.0 upscale (OpenGL renderer only). 0 = legacy present path, 1 = FSR 1.0
FSR = 1

; RCAS sharpness in stops while FSR 1.0 is active (0 = maximum, 2 = softest)
FSRSharpness = 0.2

; Legacy unsharp mask strength, used when FSR 1.0 is not active
SharpenAmount = 0.4

[Games]
c1 = /opt/carma/c1
c1demo = /opt/carma/c1demo
sp = /opt/carma/splatpack

[Cheats]
EditMode = 0
FreezeTimer = 0
GameCompleted = 0

[Sound]
Enabled = 1
SoundOptionsScreen = 1
VolumeMultiplier = 1

[Network]
AdapterName = ""
```

## Order of precedence for game directory detection:
1. Directory pointed to by DefaultGame
2. First game in the list if at least 1 game dir is specified
3. `--dir` command line argument
4. Current working directory (if `DATA/GENERAL.TXT` exists)
5. `SDL_GetPrefPath` directory

## Graphics settings (OpenGL renderer)

The OpenGL (`glrend`) renderer's FSR 1.0 upscale and sharpen settings can be
controlled with command-line flags, with the `[Graphics]` section of
`dethrace.ini`, or with the underlying `GLREND_*` environment variables. All
three control the same settings; dethrace forwards the resolved values to the
renderer through the environment before the renderer initialises.

Only settings that are specified at a given level are applied: an existing
`GLREND_*` environment variable is left untouched when neither the command line
nor `dethrace.ini` sets that setting, so the renderer defaults still apply.

Order of precedence (highest first):

1. Command-line flag
2. `[Graphics]` entry in `dethrace.ini`
3. Existing `GLREND_*` environment variable
4. Renderer default

Values must be non-negative. An invalid command-line value aborts startup with
an error; an invalid `dethrace.ini` value is ignored with a warning in the log.
These settings only affect the OpenGL renderer (`--opengl` / `Emulate3DFX`).

### Command-line flags

| Flag | Values | Description |
|---|---|---|
| `--fsr` / `--no-fsr` | | Force the AMD FidelityFX FSR 1.0 upscale on/off (same as `GLREND_FSR=1`/`0`). |
| `--fsr-sharpness=<n>` | `0`..`2` (stops) | RCAS sharpening strength while FSR 1.0 is active: `0` = maximum sharpening, higher values are softer (same as `GLREND_FSR_SHARPNESS`). Also accepted as `--fsr-sharpness <n>`. |
| `--sharpen=<n>` | `0`..`~1` | Strength of the legacy present-path unsharp mask, used when FSR 1.0 is not active (same as `GLREND_SHARPEN_AMOUNT`). Also accepted as `--sharpen <n>`. |

### `[Graphics]` section of dethrace.ini

| Key | Values | Description |
|---|---|---|
| `FSR` | `0`, `1` | FSR 1.0 upscale on/off (same as `GLREND_FSR`). |
| `FSRSharpness` | `0`..`2` (stops) | RCAS sharpening strength while FSR 1.0 is active (same as `GLREND_FSR_SHARPNESS`). |
| `SharpenAmount` | `0`..`~1` | Strength of the legacy present-path unsharp mask, used when FSR 1.0 is not active (same as `GLREND_SHARPEN_AMOUNT`). |

Key names are case-insensitive, and `fsr_sharpness` / `sharpen_amount` are
accepted as aliases for `FSRSharpness` / `SharpenAmount`.

### Renderer environment variables (lowest precedence)

These are read from the process environment by the BRender `glrend` driver, once
at startup. They are normally set through the command line or `dethrace.ini` as
described above; setting them directly only has an effect for settings that the
command line and `dethrace.ini` leave unspecified.

| Variable | Values | Default | Description |
|---|---|---|---|
| `GLREND_FSR` | `0`, `1` | `1` on OpenGL core, `0` on OpenGL ES | Enables the AMD FidelityFX FSR 1.0 (EASU + RCAS) upscale of the game's native 640x480 frame to the window resolution. `0` keeps the legacy present path (nearest upscale + unsharp mask). On OpenGL ES FSR 1.0 defaults to off and must be enabled explicitly. |
| `GLREND_FSR_SHARPNESS` | `0`..`2` (stops) | `0.2` | RCAS sharpening strength while FSR 1.0 is active: `0` = maximum sharpening, higher values are softer. |
| `GLREND_SHARPEN_AMOUNT` | `0`..`~1` | `0.4` | Strength of the legacy present-path unsharp mask. Only used when FSR 1.0 is not active (disabled, OpenGL ES, or the window is not larger than 640x480). |

FSR 1.0 is only used when the window is larger than the game's native 640x480
resolution, i.e. when there is something to upscale. When it is active, RCAS
replaces the legacy unsharp mask.
