# Anbernic Firmware Application Development Guide

This guide documents application packaging, runtime behaviour and hardware interfaces tested on Anbernic RG40XX V hardware using the original stock firmware environment referred to here as **TF1**. Most findings were verified on a first unit and independently re-confirmed on a second unit; where the two units differed, the difference is recorded below as verified per-unit behaviour.

This guide currently covers the Anbernic RG40XX V original TF1 firmware. Findings for other Anbernic models require separate verification.

Each finding is classified as one of the following:

- **Verified behaviour**: observed directly on the tested hardware.
- **Implementation detail**: code or packaging derived from verified behaviour.
- **Original-firmware lead**: identified in original-firmware application material but not yet re-confirmed on both tested units.
- **Unresolved behaviour**: not established by the available tests.

## 1. Verified environment

```text
Architecture:       aarch64
Operating system:   Ubuntu 22.04 base
Python:             3.10.12
Pillow:             9.0.1
glibc:              2.35
Kernel:             Linux 4.9.170
Display surface:    640 x 480
SDL video driver:   mali
SDL renderer:       opengles2
SDL joystick:       ANBERNIC-keys
```

The architecture, OS base, Python, Pillow, glibc, kernel and SDL video driver were re-confirmed on a second unit. The display surface is the SDL fullscreen render target used by applications. The raw framebuffer console geometry differs and is state-dependent; see Section 10.

### Pillow compatibility

TF1 uses Pillow 9.0.1. APIs introduced by newer Pillow releases are outside the verified compatibility baseline. Direct use of `Image.Resampling.LANCZOS` is not compatible with the tested environment.

```python
from PIL import Image

RESAMPLE_LANCZOS = getattr(
    getattr(Image, "Resampling", Image),
    "LANCZOS",
    getattr(Image, "LANCZOS", Image.BICUBIC),
)
```

### Stock board, language and firmware configuration leads

Original-firmware application material identifies these paths:

```text
/mnt/vendor/oem/board.ini
/mnt/vendor/oem/language.ini
```

The expected board string for this device is:

```text
RG40xxV
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

These paths and values are original-firmware leads, not yet two-unit verified facts. File contents and language indexes require validation before use. An unknown board value remains unknown rather than falling back to another device model.

```python
from pathlib import Path

def read_first_line(path, default=None):
    try:
        lines = Path(path).read_text(encoding="utf-8").splitlines()
        return lines[0].strip() if lines else default
    except (OSError, UnicodeError):
        return default
```

The exact official firmware version represented by the tests remains unresolved.

## 2. Application discovery and package layout

TF1 discovers application launchers placed directly in:

```text
/mnt/mmc/Roms/APPS
```

A folder containing only a launcher did not appear in the stock menu. A shell script placed directly in `APPS` did appear.

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

- `My_App.sh`: launcher shown by the TF1 menu.

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

`/tmp` is used for render frames, scratch files and session-only state. High-frequency generated frames are not stored under `data/`. The package layout keeps related code cohesive; separate files represent substantial subsystems rather than individual helper functions.

### Candidate stock menu icon layout

Original-firmware application material indicates this optional layout:

```text
/mnt/mmc/Roms/APPS/
|-- My_App.sh
|-- My_App/
|   `-- main.py
`-- Imgs/
    `-- My_App.png
```

The apparent convention is `Imgs/<launcher-name>.png`. Exact filename matching, dimensions, format and menu cache behaviour remain unverified. Application discovery does not depend on a custom icon.

### TF2 storage lead

Original-firmware application material identifies the probable second-card mount point as:

```text
/mnt/sdcard
```

The verified TF1 path remains `/mnt/mmc`. TF2 availability requires a mounted and accessible `/mnt/sdcard`; an existing empty directory alone does not establish that a card is mounted.

## 3. Reference launcher

Save the launcher as `/mnt/mmc/Roms/APPS/My_App.sh`.

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
            printf '%s
' "$$" > "$LOCK_PID"
            printf '%s
' "$start" > "$LOCK_START"
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

The lock records both the PID and the process start time from `/proc/<pid>/stat`, preventing a reused PID from validating a stale lock. Lock replacement is retried once; `mkdir` remains the atomic ownership operation.

Writable-directory checks select persistent application storage when available and `/tmp` fallbacks when the APPS partition is read-only. The launcher rotates `app.log` to `app.log.1` when the active log reaches 1 MiB.

`HOME`, `XDG_DATA_HOME`, `XDG_CONFIG_HOME`, `PYTHONPATH` and `LD_LIBRARY_PATH` are scoped to the application package. Ordinary exits do not call `sync`; installation and important persistent updates call it after data changes.

Launcher validation:

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

### Stock volume-controller helper

Original-firmware launchers identify `/mnt/mod/ctrl/volumeCtrl.dge`. Some launchers start this helper before a fullscreen Python application, then terminate it after the application exits. The helper may handle volume input, the stock volume heads-up display, or both. Its device access, IPC and exact lifecycle remain unresolved.

Process-name-wide termination such as `kill -9 $(pidof volumeCtrl.dge)` affects every matching instance and bypasses cleanup. A launcher that starts the helper can retain the returned PID and associate it with the process start time under `/proc/<pid>/` before terminating that specific instance.

## 4. SDL2 application baseline

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

Every SDL function called through `ctypes` requires explicit `argtypes` and `restype` declarations. This is essential for pointer-returning functions on AArch64; loading the shared library alone does not define a safe foreign-function interface.

Joystick index `0` reported `ANBERNIC-keys` on the tested units. Renderer creation first requests acceleration with present synchronisation, then falls back to software rendering if acceleration is unavailable.

Transient Pillow frames are stored under `/tmp`, for example `/tmp/My_App-screen.bmp`, and removed during normal shutdown.

### Stock SDL and PySDL2 leads

Original-firmware application material indicates:

```text
/usr/lib/libSDL2.so
/usr/lib/libSDL2-2.0.so.0.12.0
/usr/lib/python3/dist-packages/sdl2/
SDL 2.0.12
```

These exact paths and the PySDL2 installation remain unconfirmed across both units. The verified ctypes loader remains the baseline.

When PySDL2 is unavailable, pure-Python bindings can be packaged under `modules/`. Extracting `sdl2.zip` or another application archive into `/` modifies the firmware filesystem and can overwrite system files.

Stock apps have been observed requesting SDL_WINDOW_FULLSCREEN_DESKTOP | SDL_WINDOW_SHOWN. TF1 still has no verified desktop compositor. Always query actual window dimensions.

## 5. Verified physical input mapping

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

Button 13 is the normal short Menu press. TF1 emits button 8 as a separate Menu Hold event. Preserve short button 13 presses as ordinary input. If button 8 is used for navigation, remove it from held-button state after handling it rather than assuming a normal paired down/up lifetime.

Buttons 12 and 14 remain physically unassigned.

A second unit reported 14 joystick buttons, indices 0 through 13, and 6 axes through js0. Buttons 0 through 13 matched across both units, including Menu Hold and short Menu. Volume keys were not joystick buttons 15 and 16 on the second unit. Volume keys may instead be exposed through evdev `EV_KEY` events. Production code resolves the active input node by device identity rather than relying on a fixed event number.

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

In the tested SDL event buffer, hat index and value were read as:

```python
hat_index = raw[12]
hat_value = raw[13]
```

These offsets belong to the tested event representation and are not a replacement for declaring the complete SDL event structures used by production `ctypes` code.

If a button press starts an input test, arm on button down and begin capture after the activation button is released.

## 6. Analogue stick input

The physical stick press is button `9`. On the tested units, `ANBERNIC-keys` was exposed through `/dev/input/js0` and `/dev/input/event1`. SDL button and hat events work, but analogue movement was not reliably observed through the initial SDL parser. Analogue movement was read non-blockingly from `/dev/input/js0` while SDL handled display, buttons and D-pad.

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

Only axes 0 and 1 have verified physical meaning. Enumerate all axes. A practical initial dead zone is 8000; precise apps should support calibration. A 50 ms controlled sample interval was used by the reference implementation.

## 7. Verified internal-speaker audio

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

11. Exit the worker without global `SDL_Quit()`.

The parent retains a watchdog for the worker. Global `SDL_Quit()` blocked during testing after audio close. Blocking `aplay` in the graphical process became unresponsive during testing and required a forced power-off; the isolated SDL worker is the verified audio path.

Useful tests include left-only, right-only, combined output, 100 Hz through 16 kHz, 1% through 25% amplitude and 60 Hz through 315 Hz resonance checks. The validation UI uses neutral labels because physical stereo separation has not been established.

## 8. Battery telemetry and vibration lead

TF1 exposes battery telemetry through:

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
voltage_now: microvolts; divide by 1,000,000
temp: tenths of a degree Celsius; divide by 10
```

```python
from pathlib import Path

def read_sysfs(path, default="Unavailable"):
    try:
        return Path(path).read_text(encoding="utf-8").strip()
    except (OSError, UnicodeError):
        return default
```

### Candidate vibration interface

Original-firmware application material identifies:

```text
/sys/class/power_supply/axp2202-battery/moto
```

The reported values are `1` to start and `0` to stop. The interface remains unresolved on the tested units: existence, permissions, accepted values, stop behaviour, crash behaviour and emulator-rumble interaction have not been established.

A future implementation can access a verified sysfs attribute directly and write `0` from guaranteed cleanup logic. A compound `shell=True` command would make stop handling dependent on shell execution rather than application cleanup.

## 9. System information, USB power and thermals

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

Both tested states exposed no DRM connector entries, no standard backlight device, no X11 and no Wayland. Fullscreen SDL2 at 640 x 480 is the verified application rendering path; direct `/dev/fb0` output is not part of that path.

### HDMI state lead

Original-firmware material identifies:

```text
/sys/class/extcon/hdmi/state
```

Reported values are HDMI=0 and HDMI=1. This remains unconfirmed across both units.

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

HDMI connection state does not establish output geometry. Window size must be queried after an output change. HDMI resolution, scaling, hotplug safety, internal-LCD behaviour and custom-application audio remain unresolved.

### Original-firmware investigation paths

Retain the following paths in all stock heads-up display and helper-process investigations:

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

These paths cover the known stock volume helper, related controller helpers, OEM configuration, vendor binaries, process-specific runtime information and the likely framebuffer, Mali, display-composition and contiguous-memory interfaces. Discovery records the presence, permissions and role of each path without changing device or firmware state. A missing path is recorded as absent rather than created.

`/dev/fb0`, `/dev/mali0`, `/dev/disp` and `/dev/ion` are read-only investigation targets. Files under `/mnt/mod/ctrl/`, `/mnt/vendor/oem/` and `/mnt/vendor/bin/` remain unchanged. `/proc/<pid>/` is used for the selected active stock process.

### Stock heads-up display lead

/mnt/mod/ctrl/volumeCtrl.dge is the primary original-firmware lead for a system-wide heads-up display. Before implementing an independent battery popup, determine:

- Whether it displays a volume notification over custom apps and emulators.

- Whether it runs during emulation.

- Which input and display devices it opens.

- Whether it uses SDL, EGL, OpenGL ES, Mali, direct framebuffer access or another stock interface.

- Whether it communicates with `dmenu.bin` or another process.

- Whether it exposes a socket, FIFO, shared-memory object, signal interface or command-line protocol.

- Whether `/mnt/mod/ctrl` contains battery, brightness or general notification helpers.

- Whether its overlay survives emulator page flipping and HDMI output.

A second fullscreen SDL window has not been shown to provide system-wide composition. The stock helper is the active investigation target.

### Historical direct-framebuffer implementation

The older generic [Appbernic framework](https://github.com/gizmo2040/appbernic) memory-maps `/dev/fb0` as a fixed 640 x 480, 32-bit surface, applies red-and-blue channel conversion and issues hardcoded framebuffer ioctls. Those assumptions do not cover the state-dependent RG40XX V framebuffer states documented above. The code is historical prior art, not a verified RG40XX V rendering path. The repository is GPL-3.0 licensed and credits earlier graphics and input work by `macc-n`; copied or adapted source remains subject to its licence and attribution requirements.

## 11. Verified fonts and stock font lead

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

Use DejaVu Sans for labels and natural-language status, Bold for headings, Mono for measurements and Mono Bold for prominent aligned values. Liberation Sans was not present in the tested font inventory. Text fitting uses rendered pixel width rather than character count.

Original-firmware material also identifies:

```text
/mnt/vendor/bin/default.ttf
```

This stock font remains unconfirmed. The verified DejaVu paths remain preferred.

## 12. Joystick RGB configuration

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

Treat effect and enabled fields as read-only until separately verified. Before editing, verify size and integrity, clamp byte-style values to 0 through 255, preserve original bytes, write and verify the complete replacement, and restore the original on failure. The verified edit path replaces and re-reads the complete configuration file. Live LED endpoints are outside the verified interface. TF1 may reload the file only after the application exits.

## 13. Offline assets and stock resources

Required UI assets are packaged locally; the runtime path does not depend on network-hosted fonts, icons or images. A compact app may use one cohesive sprite sheet instead of many small files. Keep readable text labels so the UI remains usable if decorative assets fail.

System fonts cover the documented UI roles; an application-local font is only needed for a typeface absent from TF1. /mnt/vendor/bin/default.ttf is a stock lead, not yet a verified dependency.

The possible stock launcher-icon convention is:

```text
/mnt/mmc/Roms/APPS/Imgs/<launcher-name>.png
```

Exact dimensions and caching remain unresolved. Assets and dependencies remain inside the application package rather than the firmware root.

## 14. Verified Wi-Fi and Bluetooth

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

The older paired-devices command is unavailable. Google Pixel Buds Pro and a Nintendo Pro Controller were simultaneously paired, bonded and connected. The controller worked for normal input. BlueALSA exposes a bluealsa PCM while the Bluetooth stack runs.

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

The reference demonstrates fullscreen SDL2 and Pillow rendering, physical-layout button testing, separate Menu short and hold events, D-pad hats, direct `/dev/input/js0` analogue monitoring, range tracking, unknown-button discovery, isolated audio, screen patterns, battery, USB power, thermals, offline assets and consolidated regression tests. Analogue measurements use a 50 ms controlled sample interval. The current system-information view displays CPU and GPU telemetry; VE and DDR remain available to applications.

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

The APPS partition is VFAT, re-confirmed on a second unit. Unix ownership, executable metadata, symbolic links, case sensitivity and fully POSIX replacement semantics are unavailable as package guarantees.

`/tmp` holds scratch data. `config/`, `data/` and `logs/` hold persistent content.

The verified TF1 root is `/mnt/mmc`. Original-firmware material identifies `/mnt/sdcard` as the probable TF2 mount. Runtime storage detection distinguishes mounted, empty, removed and read-only media. Packaged code remains anchored to the launcher directory.

## 18. Local dependencies

Pure-Python dependencies belong under modules/. AArch64 shared libraries belong under lib/.

```text
Architecture:   aarch64
Python ABI:     Python 3.10
C library:      glibc 2.35 or older-compatible
```

Possible stock-provided paths:

```text
/usr/lib/python3/dist-packages/sdl2/
/usr/lib/libSDL2.so
/usr/lib/libSDL2-2.0.so.0.12.0
```

These paths require verification before use. Dependency resolution order:

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

An AArch64 build is not automatically compatible with TF1. The verified userspace ABI baseline is glibc 2.35.

## 20. Remaining unknowns

- Exact official firmware version represented by the tests.

- Whether `board.ini` reports `RG40xxV` on both units.

- Whether `language.ini` uses the reported ten-entry mapping.

- Whether `/mnt/vendor/bin/default.ttf` exists and loads through Pillow.

- Whether `/mnt/sdcard` is the consistent TF2 mount.

- Menu icon dimensions, matching rules, format and cache behaviour.

- Whether `Imgs/<launcher-name>.png` is correct on the tested firmware.

- Physical functions of buttons `12` and `14`.

- Exact evdev source and key codes for the volume keys on both units.

- SDL mapping of the power button.

- Whether reset produces a recordable event.

- Physical purpose of axes beyond `0` and `1`.

- Whether `/sys/class/extcon/hdmi/state` exists and reliably reports HDMI state.

- HDMI resolution, scaling, hotplug, internal-LCD and custom-audio behaviour.

- Whether the internal speaker preserves stereo separation.

- Whether global SDL audio shutdown can be made reliable without isolation.

- Whether RGB configuration offsets differ between TF1 releases.

- Whether the `moto` attribute controls vibration and how it behaves on failure.

- Whether stock PySDL2 exists at the reported path.

- Exact SDL version and optional SDL modules.

- Exact role and lifecycle of `volumeCtrl.dge`.

- Whether `volumeCtrl.dge` runs during emulation and draws the stock HUD.

- Which display, graphics and input devices it opens.

- Whether stock helpers expose reusable IPC.

- Whether another battery, brightness or notification helper exists.

- Whether a transparent notification can render over a running emulator without interrupting it.

- Why idle fbdev geometry differs from the SDL path beyond SDL/mali mode negotiation.

## 21. Verified application stack

### Verified stack

```text
Top-level APPS shell launcher
  -> Python 3.10.12
  -> Pillow 9.0.1
  -> Pillow-generated 640 x 480 frames under /tmp
  -> SDL2
  -> mali video driver
  -> accelerated opengles2 renderer with software fallback
  -> ANBERNIC-keys buttons and D-pad through SDL
  -> short Menu on button 13
  -> Menu Hold on button 8
  -> analogue movement through /dev/input/js0 when required
  -> audiocodec output through an isolated worker
  -> AXP2202 battery and USB telemetry
  -> Linux thermal-zone telemetry
  -> verified DejaVu fonts
  -> local offline assets
  -> rtl8821cs Wi-Fi through NetworkManager and wpa_supplicant
  -> Realtek Bluetooth through BlueZ and UART H5
  -> BlueALSA Bluetooth audio definition
```

`/mnt/mmc/Roms/APPS/My_App.sh` is the visible TF1 menu entry. `/mnt/mmc/Roms/APPS/My_App/` contains code, assets, settings, persistent data and logs.
