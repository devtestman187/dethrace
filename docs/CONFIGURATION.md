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

## Graphics environment variables (OpenGL renderer)

These are currently read from the process environment by the BRender `glrend`
driver (they are not yet wired into `dethrace.ini`) and are read once at
startup.

| Variable | Values | Default | Description |
|---|---|---|---|
| `GLREND_FSR` | `0`, `1` | `1` on OpenGL core, `0` on OpenGL ES | Enables the AMD FidelityFX FSR 1.0 (EASU + RCAS) upscale of the game's native 640x480 frame to the window resolution. `0` keeps the legacy present path (nearest upscale + unsharp mask). On OpenGL ES FSR 1.0 defaults to off and must be enabled explicitly. |
| `GLREND_FSR_SHARPNESS` | `0`..`2` (stops) | `0.2` | RCAS sharpening strength while FSR 1.0 is active: `0` = maximum sharpening, higher values are softer. |
| `GLREND_SHARPEN_AMOUNT` | `0`..`~1` | `0.4` | Strength of the legacy present-path unsharp mask. Only used when FSR 1.0 is not active (disabled, OpenGL ES, or the window is not larger than 640x480). |

FSR 1.0 is only used when the window is larger than the game's native 640x480
resolution, i.e. when there is something to upscale. When it is active, RCAS
replaces the legacy unsharp mask.
