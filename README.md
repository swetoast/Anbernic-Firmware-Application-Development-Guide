# Anbernic Stock Firmware Application Development Guide

Application packaging, runtime behaviour and hardware interfaces for Anbernic's original stock Linux firmware. The verified findings come from two Anbernic RG40XX V units; model-family findings remain compatibility leads until tested on the corresponding hardware.

In this guide, **TF1** means the first microSD-card slot and the card mounted from that slot. **TF2** means the second microSD-card slot. TF1 and TF2 are storage-slot names, not firmware names.

This guide documents the verified RG40XX V environment and records compatibility leads for related H700-based Anbernic Linux devices. RG40XX V results are `[VERIFIED]`; sibling-model compatibility is `[LEAD]` unless stated otherwise.

The tested RG40XX V baseline uses an Allwinner H700, a 64-bit Linux userspace, Linux 4.9.170, Mali-G31 graphics, a 640 x 480 panel and dual microSD slots.

## Status legend

Every finding carries one confidence tag where the distinction matters:

- `[VERIFIED]`: observed directly on the stated tested hardware. Unless another model is named, verified findings refer to the two RG40XX V units.
- `[IMPL]`: implementation derived from verified behavior.
- `[LEAD]`: identified in original-firmware material or official family documentation, but not yet confirmed on the tested units.
- `[UNRESOLVED]`: not established by the available tests.

An unknown value remains unknown rather than falling back to another device model.

## Contents

- [Quick start](#quick-start)
- [Quick reference](#quick-reference)
- [Original-firmware device family](#original-firmware-device-family)
- [1. Verified environment](#1-verified-environment)
- [2. Application discovery and package layout](#2-application-discovery-and-package-layout)
- [3. Reference launcher](#3-reference-launcher)
- [4. SDL2 application baseline](#4-sdl2-application-baseline)
- [5. Physical input mapping](#5-physical-input-mapping)
- [6. Analogue stick input](#6-analogue-stick-input)
- [7. Internal-speaker audio](#7-internal-speaker-audio)
- [8. Battery telemetry and vibration](#8-battery-telemetry-and-vibration)
- [9. System information, USB power and thermals](#9-system-information-usb-power-and-thermals)
- [10. Display, HDMI and stock heads-up display](#10-display-hdmi-and-stock-heads-up-display)
- [11. Fonts](#11-fonts)
- [12. Joystick RGB configuration](#12-joystick-rgb-configuration)
- [13. Offline assets and stock resources](#13-offline-assets-and-stock-resources)
- [14. Wi-Fi and Bluetooth](#14-wi-fi-and-bluetooth)
- [15. Reference implementation](#15-reference-implementation)
- [16. Installation and validation](#16-installation-and-validation)
- [17. VFAT and multiple-card storage](#17-vfat-and-multiple-card-storage)
- [18. Local dependencies](#18-local-dependencies)
- [19. Compiled application checks](#19-compiled-application-checks)
- [20. Remaining unknowns](#20-remaining-unknowns)
- [21. Verified RG40XX V application stack](#21-verified-rg40xx-v-application-stack)

## Quick start

1. Place a top-level shell launcher directly under `/mnt/mmc/Roms/APPS/` and place the application in a matching subdirectory.
2. Use the reference launcher in Section 3.
3. Render a 640 x 480 Pillow frame through fullscreen SDL2.
4. Install and validate:

```bash
cp -a My_App.sh /mnt/mmc/Roms/APPS/
cp -a My_App /mnt/mmc/Roms/APPS/
sync
bash -n /mnt/mmc/Roms/APPS/My_App.sh
PYTHONDONTWRITEBYTECODE=1 python3 -m py_compile /mnt/mmc/Roms/APPS/My_App/main.py
```

## Quick reference

```text
TF1 card mount:     /mnt/mmc
TF2 candidate path: /mnt/sdcard, only when mounted
APPS directory:     /mnt/mmc/Roms/APPS
Render target:      640 x 480
Video path:         SDL2 -> mali -> opengles2
Input identity:     ANBERNIC-keys
Audio output:       audiocodec
```

### Known constants

```python
SDL_INIT_VIDEO = 0x00000020
SDL_INIT_JOYSTICK = 0x00000200
SDL_WINDOW_FULLSCREEN = 0x00000001
SDL_WINDOWPOS_UNDEFINED = 0x1FFF0000
SDL_RENDERER_SOFTWARE = 0x00000001
SDL_RENDERER_ACCELERATED = 0x00000002
SDL_RENDERER_PRESENTVSYNC = 0x00000004
SDL_QUIT = 0x100
SDL_JOYAXISMOTION = 0x600
SDL_JOYHATMOTION = 0x602
SDL_JOYBUTTONDOWN = 0x603
SDL_JOYBUTTONUP = 0x604

JS_EVENT_BUTTON = 0x01
JS_EVENT_AXIS = 0x02
JS_EVENT_INIT = 0x80

FBIOGET_VSCREENINFO = 0x4600
AUDIO_S16LSB = 0x8010
```

### Input map at a glance

```text
D-pad hat 0: Up=1 Right=2 Down=4 Left=8
A=0 B=1 Y=2 X=3
L1=4 R1=5 L2=10 R2=11
Select=6 Start=7
Menu Hold=8 Stick press=9 Menu short=13
Volume Down=15 / Volume Up=16 on the first unit only
```

## Original-firmware device family

`[LEAD]` Original application material contains a shared board map for multiple Linux handhelds in the same stock-firmware family:

| Board string | Model | Layout used by stock app material |
|---|---|---|
| `RGcubexx` | RG CubeXX | 720 x 720 |
| `RG34xx` | RG34XX | 720 x 480 |
| `RG34xxSP` | RG34XX SP | 720 x 480 |
| `RG28xx` | RG28XX | rotated device path |
| `RG35xx+_P` | RG35XX Plus / RG35XX 2024 family identifier | 640 x 480 fallback |
| `RG35xxH` | RG35XX H | 640 x 480 fallback |
| `RG35xxSP` | RG35XX SP | 640 x 480 fallback |
| `RG40xxH` | RG40XX H | 640 x 480 fallback |
| `RG40xxV` | RG40XX V | 640 x 480 fallback; verified on two units |
| `RG35xxPRO` | RG35XX Pro | 640 x 480 fallback |

Anbernic publishes Linux firmware for the listed models. The official download pages establish Linux stock-firmware availability, while the shared board map is the evidence for the application-family lead. Neither source proves that paths, input mappings, display modes or helper binaries are identical across models.

Compatibility boundaries:

- Results from the two tested RG40XX V units are `[VERIFIED]` for RG40XX V.
- Shared launcher layout, board detection, language lookup, Pillow rendering and stock helper patterns are `[LEAD]` for the wider family.
- Screen geometry, rotation, button counts, analogue controls, wireless hardware, HDMI, battery sysfs names and LED controls require per-model verification.
- Probe results are attributed to the model on which the probe ran; they do not automatically promote family leads to verified cross-model behavior.

Official Linux firmware references:

- [RG40XX V](https://win.anbernic.com/download/448.html)
- [RG40XX H](https://win.anbernic.com/download/434.html)
- [RG35XX SP](https://win.anbernic.com/download/412.html)
- [RG35XX Plus](https://win.anbernic.com/download/318.html)
- [RG35XX H](https://win.anbernic.com/download/360.html)
- [RG28XX](https://win.anbernic.com/download/398.html)
- [Current Linux firmware group containing RG35XX Pro, RG CubeXX and RG34XX SP](https://win.anbernic.com/download/52.html)
- [Anbernic firmware catalogue containing RG34XX](https://win.anbernic.com/download/list_51_2/)

## 1. Verified environment

`[VERIFIED]`

```text
Architecture:       aarch64
Operating system:   Ubuntu 22.04.5 LTS
Python:             3.10.12
Pillow:             9.0.1
glibc:              2.35
Kernel:             Linux 4.9.170
Display surface:    640 x 480
SDL video driver:   mali
SDL renderer:       opengles2
SDL joystick:       ANBERNIC-keys
```

This is the authoritative runtime inventory for the two tested RG40XX V units. Architecture, OS base, Python, Pillow, glibc, kernel and SDL video driver were re-confirmed on the second unit. The display surface is the SDL fullscreen render target; raw framebuffer geometry is state-dependent and documented in Section 10.

### Pillow compatibility

The tested stock firmware uses Pillow 9.0.1. APIs introduced by newer Pillow releases are outside the verified compatibility baseline. `Image.Resampling.LANCZOS` is unavailable; the compatibility selector below uses the module-level constant.

```python
from PIL import Image

RESAMPLE_LANCZOS = getattr(
    getattr(Image, "Resampling", Image),
    "LANCZOS",
    getattr(Image, "LANCZOS", Image.BICUBIC),
)
```

### Stock board, language and firmware configuration

`[VERIFIED]` on the validation unit:

```text
/mnt/vendor/oem/board.ini     RG40xxV
/mnt/vendor/oem/language.ini  2
```

The reported stock language-index mapping is:

```text
0  zh_CN
1  zh_TW
2  en_US
3  ja_JP
4  ko_KR
5  es_LA
6  ru_RU
7  de_DE
8  fr_FR
9  pt_BR
```

The validated language index `2` therefore corresponds to `en_US` under this mapping. Treat both files as read-only configuration inputs even when their filesystem permissions are broader than an application needs. An unknown board string or language index remains unknown rather than falling back to another model or language.

```python
from pathlib import Path


def read_first_line(path, default=None):
    try:
        lines = Path(path).read_text(encoding="utf-8").splitlines()
        return lines[0].strip() if lines else default
    except (OSError, UnicodeError):
        return default
```

The validation also found `/mnt/vendor/oem/version.ini` and `/mnt/mod/ctrl/configs/ver.cfg`, but did not capture their contents. The exact public Anbernic firmware release represented by the tests remains unresolved. The validated low-level build identifiers are Linux `4.9.170`, kernel build `#14 SMP PREEMPT Thu Dec 25 12:29:54 CST 2025`, and U-Boot `2018.05` built on December 25, 2025 at 12:27:49.

## 2. Application discovery and package layout

`[VERIFIED]`

The stock firmware discovers application launchers placed directly in:

```text
/mnt/mmc/Roms/APPS
```

A folder containing only a launcher did not appear in the stock menu. A shell script placed directly in APPS did appear.

Use a top-level launcher and a matching application folder:

```text
/mnt/mmc/Roms/APPS/
|-- My_App.sh
`-- My_App/
    |-- main.py
    |-- audio_worker.py
    |-- app/
    |-- assets/
    |-- modules/
    |-- lib/
    |-- config/
    |-- data/
    |-- logs/
    `-- tests/
```

Directory roles:

- `My_App.sh`: launcher shown by the stock menu.

- `main.py`: application entry point.

- `audio_worker.py`: optional isolated SDL audio process.

- `app/`: substantial application modules.

- `assets/`: packaged images, sounds and other local assets.

- `modules/`: application-local pure-Python dependencies.

- `lib/`: application-local AArch64 shared libraries.

- `config/`: persistent settings.

- `data/`: persistent saves, durable state and intentional caches.

- `logs/`: bounded runtime logs.

- `tests/`: consolidated regression tests where the application includes them.

`/tmp` holds render frames, scratch files and session-only state. High-frequency generated frames are not stored under `data/`. Keep the source cohesive. Add a new file only when it represents a substantial separate subsystem.

### Candidate stock menu icon layout

`[LEAD]`

Original-firmware application material indicates this optional layout:

```text
/mnt/mmc/Roms/APPS/
|-- My_App.sh
|-- My_App/
|   `-- main.py
`-- Imgs/
    `-- My_App.png
```

The apparent convention is `Imgs/<launcher-name>.png`. Exact filename matching, dimensions, format and menu-cache behaviour remain unresolved. Application discovery does not depend on a custom icon.

### TF2 card storage

`[VERIFIED]` `/mnt/sdcard` exists as a directory on the validation unit, but no filesystem was mounted there during the final probe. `findmnt /mnt/sdcard` returned no mount and filesystem queries resolved to the root filesystem instead. The directory itself is therefore not evidence that TF2 is present.

Original-firmware application material still identifies `/mnt/sdcard` as the candidate TF2 mount point. Verify the mount itself before reading or writing:

```bash
findmnt -M /mnt/sdcard
```

Treat TF2 as unavailable unless the command returns a distinct mounted filesystem. Runtime handling should distinguish a mounted card from an empty directory, removed media and read-only media.

## 3. Reference launcher

`[IMPL]`

Save the launcher as `My_App.sh` in the APPS directory defined in Section 2.

```bash
#!/bin/bash

APP_DIR="/mnt/mmc/Roms/APPS/My_App"
PYTHON="/usr/bin/python3"
PERSISTENT_DATA="$APP_DIR/data"
PERSISTENT_CONFIG="$APP_DIR/config"
PERSISTENT_LOGS="$APP_DIR/logs"
TEMP_ROOT="/tmp/My_App"
LOCK_DIR="/tmp/My_App.lock"
LOCK_PID="$LOCK_DIR/pid"
LOCK_START="$LOCK_DIR/start"
LOG_MAX_BYTES=1048576

can_write_dir() {
    directory="$1"
    test_file="$directory/.write-test.$$"
    [ -d "$directory" ] || return 1
    : > "$test_file" 2>/dev/null || return 1
    rm -f "$test_file" 2>/dev/null
}

process_start() {
    pid="$1"
    [ -r "/proc/$pid/stat" ] || return 1
    awk '{print $22}' "/proc/$pid/stat" 2>/dev/null
}

lock_is_live() {
    old_pid=""
    old_start=""
    [ -r "$LOCK_PID" ] && old_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    [ -r "$LOCK_START" ] && old_start="$(cat "$LOCK_START" 2>/dev/null)"
    case "$old_pid:$old_start" in
        *[!0-9:]*|:*|*:) return 1 ;;
    esac
    [ "$(process_start "$old_pid")" = "$old_start" ]
}

acquire_lock() {
    attempts=0
    while [ "$attempts" -lt 2 ]; do
        if mkdir "$LOCK_DIR" 2>/dev/null; then
            start="$(process_start "$$")" || return 1
            printf '%s\n' "$$" > "$LOCK_PID"
            printf '%s\n' "$start" > "$LOCK_START"
            return 0
        fi
        lock_is_live && return 1
        rm -rf "$LOCK_DIR" 2>/dev/null || return 1
        attempts=$((attempts + 1))
    done
    return 1
}

cleanup() {
    current_pid=""
    current_start=""
    [ -r "$LOCK_PID" ] && current_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    [ -r "$LOCK_START" ] && current_start="$(cat "$LOCK_START" 2>/dev/null)"
    if [ "$current_pid" = "$$" ] && [ "$current_start" = "$(process_start "$$")" ]; then
        rm -rf "$LOCK_DIR" 2>/dev/null || true
    fi
}

rotate_log() {
    log_file="$1"
    [ -f "$log_file" ] || return 0
    size="$(wc -c < "$log_file" 2>/dev/null)" || return 0
    case "$size" in
        ''|*[!0-9]*) return 0 ;;
    esac
    if [ "$size" -ge "$LOG_MAX_BYTES" ]; then
        rm -f "$log_file.1" 2>/dev/null
        mv "$log_file" "$log_file.1" 2>/dev/null || true
    fi
}

mkdir -p "$PERSISTENT_DATA" "$PERSISTENT_CONFIG" "$PERSISTENT_LOGS" 2>/dev/null || true
mkdir -p "$TEMP_ROOT/home" "$TEMP_ROOT/config" "$TEMP_ROOT/data" "$TEMP_ROOT/logs" || exit 1

acquire_lock || exit 0
trap cleanup EXIT INT TERM HUP

if can_write_dir "$PERSISTENT_DATA"; then
    export HOME="$PERSISTENT_DATA"
    export XDG_DATA_HOME="$PERSISTENT_DATA"
else
    export HOME="$TEMP_ROOT/home"
    export XDG_DATA_HOME="$TEMP_ROOT/data"
fi

if can_write_dir "$PERSISTENT_CONFIG"; then
    export XDG_CONFIG_HOME="$PERSISTENT_CONFIG"
else
    export XDG_CONFIG_HOME="$TEMP_ROOT/config"
fi

if can_write_dir "$PERSISTENT_LOGS"; then
    LOG_FILE="$PERSISTENT_LOGS/app.log"
else
    LOG_FILE="$TEMP_ROOT/logs/app.log"
fi

rotate_log "$LOG_FILE"

export SDL_NOMOUSE=1
export PYTHONDONTWRITEBYTECODE=1

if [ -d "$APP_DIR/modules" ]; then
    export PYTHONPATH="$APP_DIR/modules:${PYTHONPATH:-}"
fi

if [ -d "$APP_DIR/lib" ]; then
    export LD_LIBRARY_PATH="$APP_DIR/lib:${LD_LIBRARY_PATH:-}"
fi

cd "$APP_DIR" || exit 1
"$PYTHON" "$APP_DIR/main.py" >> "$LOG_FILE" 2>&1
exit $?
```

The lock stores the PID and process start time from `/proc/<pid>/stat`; a reused PID cannot validate a stale lock. `mkdir` is the atomic ownership operation and stale-lock replacement is retried once.

Writable-directory checks select persistent storage when available and `/tmp` fallbacks when the APPS partition is read-only. `app.log` rotates to `app.log.1` at 1 MiB.

`HOME`, `XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `PYTHONPATH` and `LD_LIBRARY_PATH` are scoped to the package. Ordinary exits omit `sync`; installation and important persistent updates call it after data changes.

Validation:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

### Stock volume-controller helper

`[VERIFIED]` `/mnt/mod/ctrl/volumeCtrl.dge` is a stripped 32-bit ARM EABI5 hard-float executable using `/lib/ld-linux-armhf.so.3`. It is dynamically linked against the stock legacy SDL stack, including SDL 1.2, SDL_image 1.2, SDL_ttf 2.0 and SDL_gfx. A normal AArch64 `ldd` invocation reports `not a dynamic executable`; use `readelf -d` to inspect this 32-bit helper.

Embedded paths and strings verify that the helper knows about:

```text
/dev/disp
/dev/input/event0
/dev/input/event1
/mnt/data/dmenu/localpad.map
/sys/class/extcon/hdmi/state
/sys/class/power_supply/axp2202-battery/brightness
/sys/class/power_supply/axp2202-battery/moto
/sys/class/power_supply/axp2202-battery/nds_esckey
/sys/class/power_supply/axp2202-battery/nds_pwrkey
/sys/class/power_supply/axp2202-battery/openbor_volume
/sys/class/power_supply/axp2202-battery/voltage_now
/sys/class/power_supply/axp2202-usb/online
```

It also contains brightness-ioctl, HDMI-state, ALSA-device and input-discovery references. This establishes that it is a broader hardware-control helper, not merely an arbitrary launcher binary. Its exact lifecycle, HUD rendering behavior, communication with `dmenu.bin`, and behavior during emulation remain unresolved.

Some stock launchers start the helper before a fullscreen application and terminate it afterward. Avoid process-name-wide termination such as `kill -9 $(pidof volumeCtrl.dge)`. A launcher that starts the helper should retain its PID and process start time under `/proc/<pid>/` and terminate only the instance it owns.

## 4. SDL2 application baseline

`[VERIFIED]`

The tested graphical path uses:

```text
Python 3
Pillow-generated 640 x 480 frames
SDL2 loaded through ctypes
mali SDL video driver
opengles2 accelerated renderer
```

The mali video driver was re-confirmed on a second unit through a video-only SDL_InitSubSystem(SDL_INIT_VIDEO) probe. The accelerated renderer requires a window and was not re-probed headlessly.

```python
SDL_INIT_VIDEO = 0x00000020
SDL_INIT_JOYSTICK = 0x00000200
SDL_WINDOW_FULLSCREEN = 0x00000001
SDL_WINDOWPOS_UNDEFINED = 0x1FFF0000
SDL_RENDERER_SOFTWARE = 0x00000001
SDL_RENDERER_ACCELERATED = 0x00000002
SDL_RENDERER_PRESENTVSYNC = 0x00000004
SDL_QUIT = 0x100
SDL_JOYAXISMOTION = 0x600
SDL_JOYHATMOTION = 0x602
SDL_JOYBUTTONDOWN = 0x603
SDL_JOYBUTTONUP = 0x604
```

```python
import ctypes
import ctypes.util

candidates = [
    ctypes.util.find_library("SDL2"),
    "libSDL2-2.0.so.0",
    "/usr/lib/aarch64-linux-gnu/libSDL2-2.0.so.0",
]

for candidate in filter(None, candidates):
    try:
        sdl = ctypes.CDLL(candidate)
        break
    except OSError:
        pass
else:
    raise RuntimeError("SDL2 library not found")
```

Every SDL function called through `ctypes` requires explicit `argtypes` and `restype` declarations. Pointer-returning functions require this on AArch64; loading the library does not define the foreign-function interface.

Open joystick index 0 and verify the reported device name. The tested device reports ANBERNIC-keys. Request accelerated rendering with present synchronisation first, then retry with software rendering if creation fails.

Render transient Pillow frames under /tmp, for example /tmp/My_App-screen.bmp, and remove them during normal shutdown.

### Stock SDL and PySDL2

`[VERIFIED]` the stock PySDL2 package and stock SDL 2.0.12 libraries are present:

```text
/usr/lib/python3/dist-packages/sdl2/
/usr/lib/libSDL2.so
/usr/lib/libSDL2-2.0.so.0.12.0
```

A separate Ubuntu AArch64 SDL2 installation is also present:

```text
/usr/lib/aarch64-linux-gnu/libSDL2-2.0.so.0
SDL 2.28.5
```

Library selection is therefore significant. `ctypes.util.find_library("SDL2")` returns the unqualified name `libSDL2-2.0.so.0`, whose final target depends on runtime library search order. Use `/usr/lib/libSDL2-2.0.so.0.12.0` when the application specifically requires the validated stock SDL 2.0.12 and Mali behavior. Do not assume that every SDL lookup resolves to the same implementation.

When PySDL2 is unavailable on another unit or model, pure-Python bindings can be packaged under `modules/`. Do not extract an application archive into `/` or replace system libraries.

Stock applications have been observed requesting `SDL_WINDOW_FULLSCREEN_DESKTOP | SDL_WINDOW_SHOWN`. The tested environment has no verified desktop compositor. Always query the actual window dimensions.

## 5. Physical input mapping

`[VERIFIED]`

### D-pad

```text
Up:       hat 0, value 1
Right:    hat 0, value 2
Down:     hat 0, value 4
Left:     hat 0, value 8
```

### Buttons

```text
A:               button 0
B:               button 1
Y:               button 2
X:               button 3
L1:              button 4
R1:              button 5
Select:          button 6
Start:           button 7
Menu Hold:       button 8
Stick press:     button 9
L2:              button 10
R2:              button 11
Menu short:      button 13
Volume Down:     button 15 on the first unit only
Volume Up:       button 16 on the first unit only
```

Button 13 is the normal short Menu press. The stock firmware emits button `8` as a separate Menu Hold event. Preserve short button 13 presses as ordinary input. If button 8 is used for navigation, remove it from held-button state after handling it rather than assuming a normal paired down/up lifetime.

The physical reset and power controls are excluded from application input probing. This does not assign those controls to SDL button indexes `12` or `14`.

A second unit reported 14 joystick buttons, indices 0 through 13, and 6 axes through js0. Buttons 0 through 13 matched across both units, including Menu Hold and short Menu. Volume keys were not joystick buttons 15 and 16 on the second unit. Treat volume keys as potentially exposed through evdev EV_KEY events and identify the device by name rather than hardcoding event1.

```text
0   centred
1   up
2   right
3   up + right
4   down
6   down + right
8   left
9   up + left
12  down + left
```

The tested SDL event buffer exposed hat index and value at:

```python
hat_index = raw[12]
hat_value = raw[13]
```

These offsets match the tested event representation; production ctypes code still declares the complete SDL event structures.

If a button press starts an input test, arm on button down and begin capture after the activation button is released.

## 6. Analogue stick input

`[VERIFIED]` / `[IMPL]`

The physical stick press is button `9`. The tested inventory exposes ANBERNIC-keys through js0 and event1. SDL button and hat events work, but analogue movement was not reliably observed through the initial SDL parser. Read analogue movement non-blockingly from /dev/input/js0 while SDL continues to handle display, buttons and D-pad.

```python
import os
import struct

JS_EVENT_BUTTON = 0x01
JS_EVENT_AXIS = 0x02
JS_EVENT_INIT = 0x80

fd = os.open("/dev/input/js0", os.O_RDONLY | os.O_NONBLOCK)
packet = os.read(fd, 8)
time_ms, value, event_type, number = struct.unpack("<IhBB", packet)
base_type = event_type & ~JS_EVENT_INIT
```

Production code must buffer partial reads, process every complete packet, handle initialisation events separately, close the descriptor and avoid blocking the SDL loop.

```text
Axis 0: horizontal, left negative, right positive
Axis 1: vertical, up negative, down positive
```

Only axes 0 and 1 have verified physical meaning. Enumerate all axes. A practical initial dead zone is 8000; precise apps should support calibration. The reference implementation uses a 50 ms controlled sample interval.

## 7. Internal-speaker audio

`[VERIFIED]`

The internal SDL output device is `audiocodec`. It was re-confirmed as ALSA card 0 on a second unit. That unit also exposed ahubdam, ahubhdmi and a bluealsa PCM while Bluetooth was running.

```text
Sample rate:    48,000 Hz
Format:         0x8010, signed 16-bit little-endian
Channels:       2
SDL samples:    1,024 frames
Buffer size:    4,096 bytes
Silence byte:   0
```

A 120 ms, 440 Hz tone at 2% waveform amplitude was audible. Two-channel PCM acceptance does not prove two physical speakers or acoustic stereo separation.

Use an isolated audio worker:

1. Initialise SDL audio.

2. Enumerate outputs.

3. Select a name beginning with `audiocodec`.

4. Open with the verified desired format.

5. Queue PCM.

6. Unpause.

7. Wait for the queued byte count to reach zero or a deadline.

8. Pause.

9. Clear the queue.

10. Close the device.

111. Exit the worker without global `SDL_Quit()`.

The parent retains a watchdog. Global SDL_Quit() blocked during testing after audio close. Blocking `aplay` in the graphical process that tested path became unresponsive and required forced power-off.

Useful tests include left-only, right-only, combined output, 100 Hz through 16 kHz, 1% through 25% amplitude and 60 Hz through 315 Hz resonance checks. The validation UI uses neutral labels because physical stereo separation is not established.

## 8. Battery telemetry and vibration

The stock firmware exposes battery telemetry through:

```text
/sys/class/power_supply/axp2202-battery
```

Verified attributes:

```text
capacity
capacity_level
health
present
status
temp
voltage_now
```

All seven were re-confirmed on a second unit. Read them without modification and handle missing values.

```text
voltage_now: microvolts, divide by 1,000,000
 temp: tenths of a degree Celsius, divide by 10
```

```python
from pathlib import Path

def read_sysfs(path, default="Unavailable"):
    try:
        return Path(path).read_text(encoding="utf-8").strip()
    except (OSError, UnicodeError):
        return default
```

### Vibration interface

`[VERIFIED]` the attribute exists and was readable and writable by the root validation process:

```text
/sys/class/power_supply/axp2202-battery/moto
```

The final validation did not write to it. The reported values remain `1` to start and `0` to stop, but accepted values, physical effect, timing, stop behavior, crash cleanup and emulator-rumble interaction remain `[UNRESOLVED]`.

Do not expose vibration in a production application until an interactive test has confirmed both activation and reliable stopping. A future implementation should write directly to the verified sysfs attribute and guarantee a final `0` during cleanup. Avoid compound `shell=True` commands.

## 9. System information, USB power and thermals

`[VERIFIED]`

USB power:

```text
/sys/class/power_supply/axp2202-usb/online
/sys/class/power_supply/axp2202-usb/present
/sys/class/power_supply/axp2202-usb/voltage_now
```

Verified thermal zones:

```text
thermal_zone0: cpu_thermal_zone
thermal_zone1: gpu_thermal_zone
thermal_zone2: ve_thermal_zone
thermal_zone3: ddr_thermal_zone
thermal_zone4: axp2202-battery
```

All five names were re-confirmed in order on a second unit. Values are millidegrees Celsius; divide by 1,000. VE and DDR telemetry are exposed alongside CPU and GPU telemetry.

## 10. Display, HDMI and stock heads-up investigation

The tested framebuffer is `/dev/fb0`.

### Framebuffer geometry is state-dependent

Idle or console state, verified on a second unit:

```text
Visible resolution:    1280 x 1024
Virtual resolution:    1280 x 1024
Pixel depth:           16 bits
Stride:                2,560 bytes
Rotation:              0
State:                 0
```

SDL application-path state:

```text
Visible resolution:    640 x 480
Virtual resolution:    640 x 960
Pixel depth:           32 bits
Stride:                2,560 bytes
```

Reported panel modes:

```text
1280 x 1024 at 59 Hz
640 x 480 at 59 Hz
```

The identical stride is coincidental: 640 x 4 and 1280 x 2 both equal 2,560. Stride alone does not identify framebuffer geometry.

```python
import fcntl
import os
import struct

fd = os.open("/dev/fb0", os.O_RDONLY)
buffer = bytearray(256)
fcntl.ioctl(fd, 0x4600, buffer)
xres, yres, xres_virtual, yres_virtual, xoffset, yoffset, bpp = struct.unpack_from("<7I", buffer, 0)
os.close(fd)
```

Both tested states exposed no DRM connector entries, no standard backlight device, no X11 and no Wayland. Fullscreen SDL2 at 640 x 480 is the verified rendering path; direct `/dev/fb0` output is not part of that path.

### HDMI state

`[VERIFIED]` the HDMI state interface exists and is readable:

```text
/sys/class/extcon/hdmi/state
```

The disconnected validation state returned:

```text
HDMI=0
```

```python
from pathlib import Path


def read_hdmi_state():
    try:
        value = Path("/sys/class/extcon/hdmi/state").read_text(encoding="utf-8").strip()
    except (OSError, UnicodeError):
        return None
    if value == "HDMI=1":
        return True
    if value == "HDMI=0":
        return False
    return None
```

The connected value `HDMI=1`, hotplug behavior, output geometry, scaling, internal-LCD behavior and custom-application audio transitions remain unresolved. HDMI state alone does not establish output geometry. Query the actual window dimensions after any output change.

### Stock heads-up display lead

`[LEAD]` / `[UNRESOLVED]`

/mnt/mod/ctrl/volumeCtrl.dge is the primary original-firmware lead for a system-wide heads-up display. Before implementing an independent battery popup, determine:

- Whether it displays a volume notification over custom apps and emulators.

- Whether it runs during emulation.

- Which input and display devices it opens.

- Whether it uses SDL, EGL, OpenGL ES, Mali, direct framebuffer access or another stock interface.

- Whether it communicates with `dmenu.bin` or another process.

- Whether it exposes a socket, FIFO, shared-memory object, signal interface or command-line protocol.

- Whether `/mnt/mod/ctrl` contains battery, brightness or general notification helpers.

- Whether its overlay survives emulator page flipping and HDMI output.

A second fullscreen SDL window has not been shown to provide system-wide composition. The stock helper remains the active investigation target.

`[VERIFIED]` The running `dmenu.bin` stock-menu process had `/dev/fb0`, `/dev/ion`, `/dev/disp`, `/dev/input/event0`, `/dev/input/event1`, `/dev/snd/pcmC0D0p`, `/dev/ttyS5` and `/mnt/vendor/bin/default.ttf` open. Its mappings included `/dev/fb0`, deleted DMA buffers and vendor SDL 1.2 libraries. This establishes a direct framebuffer/display path using ION-backed DMA buffers and legacy SDL 1.2 components for the stock menu. It does not establish a reusable overlay API for custom applications.

Screen validation in the reference implementation includes solid colours, greyscale, colour bars, gradients, checkerboards, one-pixel lines, borders, corner markers, a centre crosshair and cycling pixel-inspection fields. Brightness and resolution control interfaces remain unresolved.

### Original-firmware investigation paths

`[VERIFIED]` These paths exist on the validation unit and remain relevant to stock HUD and helper-process investigations:

```text
/mnt/mod/ctrl/volumeCtrl.dge
/mnt/mod/ctrl/
/mnt/vendor/oem/
/mnt/vendor/bin/
/proc/<pid>/
/dev/fb0
/dev/mali0
/dev/disp
/dev/ion
```

Validated permissions were:

```text
/dev/fb0    root:video  0660
/dev/mali0  root:root   0666
/dev/disp   root:root   0600
/dev/ion    root:root   0600
```

Non-root applications cannot assume access to `/dev/disp` or `/dev/ion`. Discovery records presence, permissions, open descriptors, mapped libraries and IPC without changing firmware or device state.

### Historical direct-framebuffer prior art

The older [Appbernic framework](https://github.com/gizmo2040/appbernic) memory-maps `/dev/fb0` as a fixed 640 x 480, 32-bit surface, applies red/blue channel conversion and issues hardcoded framebuffer ioctls. Those assumptions do not cover the state-dependent RG40XX V framebuffer states documented here, so the implementation is prior art rather than a verified RG40XX V rendering path.

The repository contains a `GPL-3.0.txt` file and its README credits earlier graphics and input work by `macc-n`. Copied or adapted source therefore requires licence and attribution review.

## 11. Fonts

Verified usable fonts:

```text
/usr/share/fonts/truetype/dejavu/DejaVuSans.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSans-Bold.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSansMono-Bold.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSerif.ttf
/usr/share/fonts/truetype/dejavu/DejaVuSerif-Bold.ttf
/usr/share/fonts/TTF/DejaVuSansMono.ttf
```

All seven were re-confirmed on a second unit and loaded with Pillow at sizes 10, 12, 14, 16, 22 and 36.

DejaVu Sans covers labels and natural-language status, Bold covers headings, Mono covers measurements and Mono Bold covers prominent aligned values. Liberation Sans was not present in the tested inventory. Text fitting uses rendered pixel width rather than character count.

The stock font is also verified:

```text
/mnt/vendor/bin/default.ttf
```

It loaded successfully through Pillow and was held open by the running stock menu. A same-size copy was present at `/mnt/mod/ctrl/configs/default.ttf`. Prefer `/mnt/vendor/bin/default.ttf` when matching the stock interface, while the verified DejaVu fonts remain suitable application defaults.

## 12. Joystick RGB configuration

`[VERIFIED]`

The tested configuration is:

```text
/mnt/data/dmenu/mculed_attr.ini
```

```text
Size:       184 bytes
Structure:  46 little-endian unsigned 32-bit words
Integrity:  word 45
```

The structure was re-confirmed on a second unit. Identified fields:

```text
word 27    foreground red
word 28    foreground green
word 29    foreground blue
word 30    brightness
word 42    background red
word 43    background green
word 44    background blue
```

Effect and enabled fields remain outside the verified edit interface. The verified file update validates size and integrity, clamps byte-style values to 0 through 255, preserves the original bytes, writes and re-reads the complete replacement, and restores the original after failed verification. Live LED endpoints are not part of the verified interface. The stock menu may reload the file only after the application exits.

## 13. Offline assets and stock resources

Required UI assets are packaged locally; runtime does not depend on network-hosted fonts, icons or images. A compact application may use one cohesive sprite sheet instead of many small files. Readable text labels preserve usability when decorative assets cannot be loaded.

Verified system fonts cover the documented UI roles. `/mnt/vendor/bin/default.ttf` is also verified and may be used when an application intentionally wants the stock typeface.

The possible stock launcher-icon convention is:

```text
/mnt/mmc/Roms/APPS/Imgs/<launcher-name>.png
```

Exact dimensions and caching remain unresolved. Assets and dependencies stay inside the application package rather than the firmware root.

## 14. Wi-Fi and Bluetooth

`[VERIFIED]`

Wi-Fi interfaces:

```text
wlan0
wlan1
```

Both use rtl8821cs, re-confirmed on a second unit. The tested environment exposes 2.4 GHz and 5 GHz support. wlan0 connected in managed mode on 5 GHz and a saved profile reconnected after reboot.

Active stack:

```text
NetworkManager
wpa_supplicant
```

Available tools:

```text
ip
iw
iwconfig
rfkill
wpa_cli
wpa_supplicant
nmcli
```

The default gateway was reachable. A single DNS lookup returned no result, so DNS reliability was not established.

Bluetooth exposes hci0 through a Realtek UART controller reporting HCI 4.1. The active stack includes bluetoothd, rtk_hciattach and rtl_btlpm. Available tools include bluetoothctl, hciconfig, hcitool and rfkill.

The tested bluetoothctl supports:

```text
bluetoothctl devices Paired
bluetoothctl devices Connected
bluetoothctl devices Bonded
```

The older paired-devices command is unavailable. Google Pixel Buds Pro and a Nintendo Pro Controller were previously verified as simultaneously paired, bonded and connected, and the controller worked for normal input. During the final validation both devices were paired and bonded but not connected. A `bluealsa` PCM definition was installed, but no `bluealsa` process was running during that probe; do not equate PCM availability with an active Bluetooth-audio service.

## 15. Reference implementation

```text
/mnt/mmc/Roms/APPS/
|-- Diagnostics.sh
`-- Diagnostics/
    |-- main.py
    |-- audio_worker.py
    |-- assets/
    |   `-- diagnostics-icons.png
    |-- data/
    |-- logs/
    `-- tests/
        `-- test_diagnostics.py
```

The reference demonstrates fullscreen SDL2 and Pillow rendering, physical-layout button testing, separate Menu short and hold events, D-pad hats, direct js0 analogue monitoring, controlled sampling, range tracking, unknown-button discovery, isolated audio, screen patterns, battery, USB power, thermals, offline assets and consolidated regression tests.

Generated frames remain under `/tmp`, not `data/`.

## 16. Installation and validation

```bash
cp -a My_App.sh /mnt/mmc/Roms/APPS/
cp -a My_App /mnt/mmc/Roms/APPS/
sync
```

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
PYTHONDONTWRITEBYTECODE=1 python3 -m py_compile /mnt/mmc/Roms/APPS/My_App/main.py
ls -lah /mnt/mmc/Roms/APPS/My_App.sh
ls -lah /mnt/mmc/Roms/APPS/My_App/
```

Run sync after installation, not after every ordinary exit.

## 17. VFAT and multiple-card storage

`[VERIFIED]`

The APPS partition is VFAT, re-confirmed on a second unit. Unix ownership, executable metadata, symbolic links, case sensitivity and fully POSIX replacement semantics are unavailable as package guarantees.

Use /tmp for scratch data. Use config/, data/ and logs/ only for persistent content. Avoid unnecessary writes.

The verified TF1 card mount is `/mnt/mmc`. `/mnt/sdcard` exists on the validation unit but was not a mount point during the final probe; it resolved to the root filesystem. Original-firmware material identifies it only as the candidate TF2 mount. Require a successful `findmnt -M /mnt/sdcard` result before treating it as TF2. Keep packaged code anchored to the launcher directory rather than searching both cards.

## 18. Local dependencies

Pure-Python dependencies belong under modules/. AArch64 shared libraries belong under lib/.

```text
Architecture:   aarch64
Python ABI:     Python 3.10
C library:      glibc 2.35 or older-compatible
```

Verified stock-provided paths:

```text
/usr/lib/python3/dist-packages/sdl2/
/usr/lib/libSDL2.so
/usr/lib/libSDL2-2.0.so.0.12.0
```

These paths and SDL 2.0.12 were confirmed on the validation unit. A separate SDL 2.28.5 installation exists under `/usr/lib/aarch64-linux-gnu/`, so explicit library selection is required when stock Mali behavior matters. Dependency resolution order:

1. Verified system-provided library.

2. Application-local pure-Python module under `modules/`.

3. Application-local compatible AArch64 shared library under `lib/`.

Application dependencies remain package-local. Extraction into `/`, package installation from the launcher, system-library replacement and writes to system Python directories are outside the documented package model.

## 19. Compiled application checks

```bash
file my-app
readelf -l my-app | grep interpreter
readelf -d my-app | grep NEEDED
ldd my-app
```

An AArch64 build is not automatically compatible with the tested stock firmware. The verified userspace ABI baseline is glibc 2.35.

## 20. Remaining unknowns

`[UNRESOLVED]` Items still requiring targeted or interactive validation:

### Firmware and package discovery

- [ ] Exact public Anbernic firmware release represented by the tests.
- [ ] Contents and meaning of `/mnt/vendor/oem/version.ini` and `/mnt/mod/ctrl/configs/ver.cfg`.
- [ ] Actual TF2 mount point with a second card inserted.
- [ ] Menu icon path, filename matching, dimensions, format and cache behavior.

### Input and audio

- [ ] Exact evdev source and key codes for both volume keys.
- [ ] Physical purpose of analogue axes beyond `0` and `1`.
- [ ] Internal-speaker channel routing and physical stereo separation.
- [ ] Global SDL audio shutdown behavior without worker isolation.

### Display, HDMI and vibration

- [ ] Connected HDMI value, resolution, scaling, hotplug, internal-LCD behavior and audio transitions.
- [ ] Vibration activation and stop semantics through the verified `moto` attribute, including failure cleanup.
- [ ] Reason for the idle and SDL framebuffer-mode difference beyond observed SDL/Mali mode negotiation.

### Stock helpers and overlay path

- [ ] `volumeCtrl.dge` lifecycle, exact functions and behavior during emulation.
- [ ] Whether `volumeCtrl.dge` renders a HUD itself or delegates rendering to another process.
- [ ] Communication between `volumeCtrl.dge`, `dmenu.bin` and other stock processes.
- [ ] Reusable sockets, FIFOs, shared memory, signals or command interfaces.
- [ ] Transparent notification rendering over an active emulator without interrupting page flipping.
- [ ] Whether the unusually large set of `/proc/<pid>/mounts` descriptors observed in `dmenu.bin` is intentional stock behavior or a resource leak.

## 21. Verified RG40XX V application stack

```text
Stock APPS launcher
  -> Python 3.10.12 and Pillow 9.0.1
  -> SDL2 using mali and opengles2
  -> SDL buttons and D-pad plus /dev/input/js0 analogue input
  -> isolated SDL audio worker using audiocodec
  -> power, battery and thermal telemetry through sysfs
```

The visible stock-menu entry is `/mnt/mmc/Roms/APPS/My_App.sh`; the matching application directory contains code, package-local dependencies, assets and persistent application data.
