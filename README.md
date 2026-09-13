# RG40XX V Stock Firmware Application Development Guide

This guide documents application packaging, runtime behaviour and hardware interfaces tested on Anbernic RG40XX V hardware using the original stock firmware environment referred to here as **TF1**. Most findings were verified on a first unit and independently re-confirmed on a second unit; where the two units differed, the difference is recorded below as verified per-unit behaviour.

This guide distinguishes between:

- **Verified behaviour**, observed directly on the tested device.
- **Recommended practice**, based on those verified results.
- **Stock-firmware leads**, found in original-firmware application material but not yet re-confirmed on both tested units.
- **Unresolved behaviour**, which should not be assumed by applications.

These findings apply to the tested original TF1 environment. Other firmware releases may use different paths, libraries, mappings, display initialisation or audio behaviour. Findings from KNULLI, muOS, Batocera, Stock OS MOD and other offshoots are outside the scope of this guide unless separately verified on original TF1.

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

TF1 uses Pillow 9.0.1. Do not require APIs introduced by newer Pillow releases. In particular, direct use of `Image.Resampling.LANCZOS` is not compatible with the tested environment.

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

These paths and values are stock-firmware leads, not yet two-unit verified facts. Validate file contents and index ranges before use. Do not silently identify an unknown device as another Anbernic model.

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

Use `/tmp` for render frames, scratch files and session-only state. Do not write high-frequency generated frames into `data/`. Keep the source cohesive. Add a new file only when it represents a substantial separate subsystem.

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

The apparent convention is `Imgs/<launcher-name>.png`. Exact filename matching, dimensions, format and menu cache behaviour remain unverified. Applications must remain usable without a custom icon.

### TF2 storage lead

Original-firmware application material identifies the probable second-card mount point as:

```text
/mnt/sdcard
```

The verified TF1 path remains `/mnt/mmc`. Before using TF2, confirm the path exists, is mounted and is accessible. An existing empty directory is not proof that a card is mounted.

## 3. Recommended launcher

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

can_write_dir() {
    directory="$1"
    test_file="$directory/.write-test.$$"
    [ -d "$directory" ] || return 1
    : > "$test_file" 2>/dev/null || return 1
    rm -f "$test_file" 2>/dev/null
}

acquire_lock() {
    if mkdir "$LOCK_DIR" 2>/dev/null; then
        printf '%s\n' "$$" > "$LOCK_PID"
        return 0
    fi

    old_pid=""
    [ -r "$LOCK_PID" ] && old_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    case "$old_pid" in
        ''|*[!0-9]*) ;;
        *)
            if kill -0 "$old_pid" 2>/dev/null; then
                return 1
            fi
            ;;
    esac

    rm -rf "$LOCK_DIR" 2>/dev/null || return 1
    mkdir "$LOCK_DIR" 2>/dev/null || return 1
    printf '%s\n' "$$" > "$LOCK_PID"
}

cleanup() {
    current_pid=""
    [ -r "$LOCK_PID" ] && current_pid="$(cat "$LOCK_PID" 2>/dev/null)"
    if [ "$current_pid" = "$$" ]; then
        rm -rf "$LOCK_DIR" 2>/dev/null || true
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

The lock prevents multiple fullscreen instances. Writable-directory checks allow the app to start with temporary state if the APPS partition is read-only.

Bound persistent log growth. The launcher above appends to `app.log`, so either implement rotation before launch or make the application responsible for rotation. Do not claim logs are bounded unless the package actually enforces a limit.

Do not run `sync` after every normal exit. Use it after installation or an important persistent update when data changed.

```bash
bash -n /mnt/mmc/Roms/APPS/My_App.sh
sed -i 's/\r$//' /mnt/mmc/Roms/APPS/My_App.sh
```

Do not install packages, upgrade components or modify firmware from the launcher.

### Stock volume-controller helper

Original-firmware launchers identify:

```text
/mnt/mod/ctrl/volumeCtrl.dge
```

Some stock launchers start this helper before a fullscreen Python application. It may handle volume buttons, the stock volume heads-up display, or both. Its exact role is unresolved.

Do not use `kill -9 $(pidof volumeCtrl.dge)`. That can kill instances not started by the current launcher and prevents clean shutdown. If later testing proves the helper is required, retain the exact PID started by the launcher and terminate only that process with a normal signal first.

Do not add the helper to the recommended launcher until testing confirms that it is required, does not conflict with SDL input, avoids duplicate instances and returns cleanly to TF1.

## 4. SDL2 application baseline

The tested graphical path uses:

```text
Python 3
Pillow-generated 640 x 480 frames
SDL2 loaded through ctypes
mali SDL video driver
opengles2 accelerated renderer
```

The `mali` video driver was re-confirmed on a second unit through a video-only `SDL_InitSubSystem(SDL_INIT_VIDEO)` probe. The accelerated renderer requires a window and was not re-probed headlessly.

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

Open joystick index `0` and verify the reported device name. The tested device reports `ANBERNIC-keys`. Request accelerated rendering with present synchronisation first, then retry with software rendering if creation fails.

Render transient Pillow frames under `/tmp`, for example `/tmp/My_App-screen.bmp`, and remove them during normal shutdown.

### Stock SDL and PySDL2 leads

Original-firmware application material indicates:

```text
/usr/lib/libSDL2.so
/usr/lib/libSDL2-2.0.so.0.12.0
/usr/lib/python3/dist-packages/sdl2/
SDL 2.0.12
```

These exact paths and the PySDL2 installation remain unconfirmed across both units. The verified ctypes loader remains the baseline.

If PySDL2 is unavailable, package pure-Python bindings under `modules/`. Never extract `sdl2.zip` or any application archive into `/`. Root extraction can overwrite firmware files and violates the package-local dependency model.

Stock apps have been observed requesting `SDL_WINDOW_FULLSCREEN_DESKTOP | SDL_WINDOW_SHOWN`. TF1 still has no verified desktop compositor. Always query actual window dimensions.

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

Button `13` is the normal short Menu press. TF1 emits button `8` as a separate Menu Hold event. Preserve short button `13` presses as ordinary input. If button `8` is used for navigation, remove it from held-button state after handling it rather than assuming a normal paired down/up lifetime.

Buttons `12` and `14` remain physically unassigned.

A second unit reported `14` joystick buttons, indices `0` through `13`, and `6` axes through `js0`. Buttons `0` through `13` matched across both units, including Menu Hold and short Menu. Volume keys were not joystick buttons `15` and `16` on the second unit. Treat volume keys as potentially exposed through evdev `EV_KEY` events and identify the device by name rather than hardcoding `event1`.

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

If a button press starts an input test, arm on button down and begin capture after the activation button is released.

## 6. Analogue stick input

The physical stick press is `button 9`. The tested inventory exposes `ANBERNIC-keys` through `js0` and `event1`. SDL button and hat events work, but analogue movement was not reliably observed through the initial SDL parser. Read analogue movement non-blockingly from `/dev/input/js0` while SDL continues to handle display, buttons and D-pad.

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

Only axes `0` and `1` have verified physical meaning. Enumerate all axes. A practical initial dead zone is `8000`; precise apps should support calibration. Diagnostics uses a 50 ms controlled sample interval.

## 7. Verified internal-speaker audio

The internal SDL output device is `audiocodec`. It was re-confirmed as ALSA card `0` on a second unit. That unit also exposed `ahubdam`, `ahubhdmi` and a `bluealsa` PCM while Bluetooth was running.

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

The parent must retain a watchdog. Global `SDL_Quit()` blocked during testing after audio close. Do not use blocking `aplay` from the graphical process; that tested path became unresponsive and required forced power-off.

Useful tests include left-only, right-only, combined output, 100 Hz through 16 kHz, 1% through 25% amplitude and 60 Hz through 315 Hz resonance checks. Use neutral labels unless physical stereo separation is verified.

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

### Candidate vibration interface

Original-firmware application material identifies:

```text
/sys/class/power_supply/axp2202-battery/moto
```

The reported values are `1` to start and `0` to stop. This is unresolved on the tested units. Do not write to it until existence, permissions, accepted values, stop behaviour, crash behaviour and emulator-rumble interaction are verified.

If verified later, open the sysfs attribute directly. Do not use `shell=True` with a compound echo command. Always guarantee that `0` is written during cleanup, keep pulses short and rate-limit them.

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

All five names were re-confirmed in order on a second unit. Values are millidegrees Celsius; divide by `1,000`. Diagnostics currently displays CPU and GPU telemetry. VE and DDR remain available to applications.

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

The identical stride is coincidental: `640 x 4` and `1280 x 2` both equal `2,560`. Never use stride alone to infer geometry.

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

Both tested states exposed no DRM connector entries, no standard backlight device, no X11 and no Wayland. Use fullscreen SDL2 at 640 x 480 instead of writing directly to `/dev/fb0`.

### HDMI state lead

Original-firmware material identifies:

```text
/sys/class/extcon/hdmi/state
```

Reported values are `HDMI=0` and `HDMI=1`. This remains unconfirmed across both units.

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

Do not assume HDMI keeps the internal geometry. Re-query window size and handle missing or unknown values. HDMI resolution, scaling, hotplug safety, internal-LCD behaviour and custom-app audio remain unresolved.

### Stock heads-up display lead

`/mnt/mod/ctrl/volumeCtrl.dge` is the primary original-firmware lead for a system-wide heads-up display. Before implementing an independent battery popup, determine:

- Whether it displays a volume notification over custom apps and emulators.
- Whether it runs during emulation.
- Which input and display devices it opens.
- Whether it uses SDL, EGL, OpenGL ES, Mali, direct framebuffer access or another stock interface.
- Whether it communicates with `dmenu.bin` or another process.
- Whether it exposes a socket, FIFO, shared-memory object, signal interface or command-line protocol.
- Whether `/mnt/mod/ctrl` contains battery, brightness or general notification helpers.
- Whether its overlay survives emulator page flipping and HDMI output.

Do not assume a second fullscreen SDL window can behave like an Android overlay. Probe the stock helper first.

Recommended screen tests remain solid colours, greyscale, colour bars, gradients, checkerboards, one-pixel lines, borders, corner markers, a centre crosshair and cycling pixel-inspection fields. Do not implement brightness or resolution changes before a safe interface is verified.

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

All seven were re-confirmed on a second unit and loaded with Pillow at sizes `10`, `12`, `14`, `16`, `22` and `36`.

Use DejaVu Sans for labels and natural-language status, Bold for headings, Mono for measurements and Mono Bold for prominent aligned values. Do not assume Liberation Sans exists. Measure rendered pixel width instead of truncating by character count.

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

Treat effect and enabled fields as read-only until separately verified. Before editing, verify size and integrity, clamp byte-style values to `0` through `255`, preserve original bytes, write and verify the complete replacement, and restore the original on failure. Do not write unverified data to a live LED endpoint. TF1 may reload the file only after the app exits.

## 13. Offline assets and stock resources

Bundle required UI assets locally. Do not depend on network-hosted fonts, icons or images. A compact app may use one cohesive sprite sheet instead of many small files. Keep readable text labels so the UI remains usable if decorative assets fail.

Do not bundle fonts already available in the verified environment unless the application requires another typeface. `/mnt/vendor/bin/default.ttf` is a stock lead, not yet a verified dependency.

The possible stock launcher-icon convention is:

```text
/mnt/mmc/Roms/APPS/Imgs/<launcher-name>.png
```

Exact dimensions and caching remain unresolved. Never extract assets or dependencies into the firmware root.

## 14. Verified Wi-Fi and Bluetooth

Wi-Fi interfaces:

```text
wlan0
wlan1
```

Both use `rtl8821cs`, re-confirmed on a second unit. The tested environment exposes 2.4 GHz and 5 GHz support. `wlan0` connected in managed mode on 5 GHz and a saved profile reconnected after reboot.

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

Bluetooth exposes `hci0` through a Realtek UART controller reporting HCI 4.1. The active stack includes `bluetoothd`, `rtk_hciattach` and `rtl_btlpm`. Available tools include `bluetoothctl`, `hciconfig`, `hcitool` and `rfkill`.

The tested `bluetoothctl` supports:

```text
bluetoothctl devices Paired
bluetoothctl devices Connected
bluetoothctl devices Bonded
```

The older `paired-devices` command is unavailable. Google Pixel Buds Pro and a Nintendo Pro Controller were simultaneously paired, bonded and connected. The controller worked for normal input. BlueALSA exposes a `bluealsa` PCM while the Bluetooth stack runs.

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

The reference demonstrates fullscreen SDL2 and Pillow rendering, physical-layout button testing, separate Menu short and hold events, D-pad hats, direct `js0` analogue monitoring, controlled sampling, range tracking, unknown-button discovery, isolated audio, screen patterns, battery, USB power, thermals, offline assets and consolidated regression tests.

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

Run `sync` after installation, not after every ordinary exit.

## 17. VFAT and multiple-card storage

The APPS partition is VFAT, re-confirmed on a second unit. Do not rely on Unix ownership, executable metadata, symbolic links, case sensitivity or fully POSIX replacement semantics.

Use `/tmp` for scratch data. Use `config/`, `data/` and `logs/` only for persistent content. Avoid unnecessary writes.

The verified TF1 root is `/mnt/mmc`. `/mnt/sdcard` is the probable TF2 mount from stock material. Before using TF2, confirm it is mounted and accessible. Handle missing, empty, removed and read-only cards. Keep packaged code anchored to the launcher directory rather than searching both cards.

## 18. Local dependencies

Pure-Python dependencies belong under `modules/`. AArch64 shared libraries belong under `lib/`.

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

Verify before depending on them. Prefer dependencies in this order:

1. Verified system-provided library.
2. Application-local pure-Python module under `modules/`.
3. Application-local compatible AArch64 shared library under `lib/`.

Do not extract archives into `/`, install packages from the launcher, replace system libraries or add files to system Python directories.

## 19. Compiled application checks

```bash
file my-app
readelf -l my-app | grep interpreter
readelf -d my-app | grep NEEDED
ldd my-app
```

An AArch64 build is not automatically compatible. Do not require glibc newer than `2.35`.

## 20. Safe development workflow

1. Keep the last working package as a rollback copy.
2. Change one substantial subsystem at a time.
3. Validate shell and Python syntax before copying.
4. Copy the launcher and matching application folder into `APPS`.
5. Run `sync` after installation.
6. Launch from the TF1 menu.
7. Inspect bounded application-local logs.
8. Confirm a clean return to the stock menu.
9. Use isolated workers and parent watchdogs for operations that may block.
10. Keep modules cohesive and avoid unnecessary one-function files.

Do not initially:

- Replace `dmenu.bin`.
- Edit stock launcher scripts.
- Install into `/mnt/vendor`.
- Stop the stock menu process.
- Write directly to `/dev/fb0`.
- Hardcode an evdev event number when the device can be identified by name.
- Extract application archives into `/`.
- Depend on `/mnt/sdcard` without verifying TF2 is mounted.
- Start or kill every `volumeCtrl.dge` process by name.
- Write to the vibration attribute before its behaviour is verified.
- Assume the stock volume helper provides a public notification API.
- Assume `HDMI=1` preserves internal display geometry.

Do not copy, replace or modify helpers under `/mnt/mod/ctrl`. If a helper is later verified as required, retain its exact PID and terminate only that instance.

## 21. Remaining unknowns

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

## 22. Verified application stack and stock-firmware leads

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

### Original-firmware leads awaiting confirmation

```text
Board identity:          /mnt/vendor/oem/board.ini
System language:         /mnt/vendor/oem/language.ini
Stock default font:      /mnt/vendor/bin/default.ttf
HDMI state:              /sys/class/extcon/hdmi/state
Probable TF2 mount:      /mnt/sdcard
Probable vibration:      /sys/class/power_supply/axp2202-battery/moto
Stock controller dir:    /mnt/mod/ctrl
Stock volume helper:     /mnt/mod/ctrl/volumeCtrl.dge
Possible stock PySDL2:   /usr/lib/python3/dist-packages/sdl2/
Possible menu icon:      Imgs/<launcher-name>.png
```

Use `/mnt/mmc/Roms/APPS/My_App.sh` as the visible TF1 menu entry and `/mnt/mmc/Roms/APPS/My_App/` for code, assets, settings, persistent data and logs.
