# prober

## Web app available https://fusionfpv.com/prober

Tool for flashing and configuring Bluejay ESC firmware over BLHeli 4-way passthrough (MSP). Supports single ESCs, 4-in-1 ESCs and AIO boards, as well as direct C2 programming of SiLabs MCUs using an Arduino.

Written in C++ with a standalone CLI and a Qt GUI for Windows and Linux. The GUI shows flashing progress and status for each ESC, and can be used to read and edit settings.

Download the native application from [Releases](https://github.com/leszczy/prober/releases), or use the [web version](https://fusionfpv.com/prober).

## Installation

### Windows

There are two versions available in [Releases](https://github.com/leszczy/prober/releases).

- Setup, which installs prober, creates shortcuts, adds `prober` to system PATH.
- Portable, which to use you must extract the ZIP and run the application from the extracted folder, installing VC++ Redistributable if necessary.

The GUI checks for updates on startup.

### Linux

Download the Linux archive from [Releases](https://github.com/leszczy/prober/releases) and extract it:

```sh
tar -xzf prober-x.x.x-linux.tar.gz
cd prober-x.x.x
./prober --list-ports
./prober_gui
```

If you cannot access serial ports, add your user to the appropriate group, then log out and back in:

```sh
# Arch
sudo usermod -aG uucp $USER
# Ubuntu/Debian
sudo usermod -aG dialout $USER
```

## Flashing and configuration

The CLI can flash a local Intel HEX file or select the correct firmware from the bundled Bluejay releases using the ESC identity. Settings are preserved by default and can also be erased, migrated or changed in the same run.

Firmware identity, version, layout, MCU and Bluejay settings can be read without flashing. The settings use the same field names as esc-configurator. Individual settings can be written separately, including timing, demag compensation, motor direction, PWM frequency, braking, EDT, beacon, temperature protection, power rating, LED control and the startup melody region.

### AIO's

Flight controllers can use either 0-based or 1-based ESC indexing in passthrough mode. prober wakes the ESC bootloaders using both ranges of selection values before determining the mapping. A quick check followed by a full probe is used to settle on a mapping before flashing or reading settings.

If an ESC responds to direct addressing but not to index selection, prober tries to wake it directly, resets it and retries selection. It can also exit and re-enter 4-way mode if needed. This recovery is attempted during probing, before flashing and when selecting an ESC to flash.

The `ExitBootloader` command tries the two-byte reset format with the mode byte first. The working format is remembered for the rest of the session so the ESC can release the bus without unnecessary retries. After flashing each ESC, all channels are woken again before moving on to the next one.

Use `--skip-missing` to continue a batch if an ESC still cannot be reached after recovery attempts.

### Probing

Use `--probe-escs` to check which ESCs can be reached with the selected mapping, or `--probe-sweep` to check all selection values from 0 to 255. The probe can also try direct addresses from `0x0004` to `0x0008` for ESCs that do not respond to index selection.

Probe reports include the device signature, mapping, number of attempts and whether direct addressing or index selection was used. Use `--trace` to log the individual 4-way transactions, reset formats, cached capabilities and recovery attempts.

### C2 flashing

An Arduino Uno or Nano can be used as a C2 interface to flash Bluejay directly onto a SiLabs MCU, without FC passthrough. The interface firmware can be uploaded with `--c2-install` using avrdude. C2 mode supports interface detection, reading the target's device ID and revision, erasing, writing a HEX file and resetting the target. See the C2 options below.

## CLI reference

### Connection

| Flag | Default | Description |
|------|---------|-------------|
| `--port PORT` | auto-detect | Serial port |
| `--baud N` | 115200 | Baud rate |
| `--timeout S` | 1.0 | 4-way response timeout (seconds) |
| `--delay S` | 0.0 | Inter-byte delay |
| `--safe` | off | Use a 6 second timeout and 0.01 second delay for BB51 ESCs |
| `--list-ports` | | List available serial ports and exit |

### ESC selection

| Flag | Description |
|------|-------------|
| `--index N` | Flash/read ESC N (1-based) |
| `--all` | Flash/read all ESCs |
| `--mapping MODE` | `auto`, `strict-0-based`, `strict-1-based`, `prefer-0-based`, `prefer-1-based` |
| `--skip-missing` | Continue batch even if some ESCs are unreachable |

### Flashing

| Flag | Default | Description |
|------|---------|-------------|
| `--hex PATH` | required | Hex file path, or `auto` for bundled Bluejay |
| `--verify [full\|fast\|off]` | off | Verify by byte (`full`), by page CRC (`fast`), or skip (`off`) |
| `--erase-eeprom` | off | Erase EEPROM (settings) during flash |
| `--full-erase-app` | off | Erase full application area before write |
| `--full-erase-entire-app` | off | Erase entire application region |
| `--dry-run` | off | Simulate flash without writing |
| `--verify-all-bytes` | off | Verify every byte (not just written pages) |
| `--assume-sig SIG` | | Override device signature detection |
| `--settings MODE` | preserve | `preserve`, `erase`, or `migrate` |
| `--set KEY=VALUE` | | Apply setting after flash (repeatable) |
| `--bluejay-version VER` | 0.21.0 | Bluejay version for `--hex auto` |
| `--bluejay-dir PATH` | | Override firmware search directory |

### Reading and writing settings

| Flag | Default | Description |
|------|---------|-------------|
| `--read` | | Read and display ESC settings |
| `--read-rounds N` | 2 | Number of read rounds |
| `--read-round-sleep-ms N` | 150 | Delay between read rounds (ms) |
| `--write-settings` | | Write settings without reflashing |
| `--set KEY=VALUE` | | Setting to write (repeatable) |

### Probing and debugging

| Flag | Description |
|------|-------------|
| `--probe-escs` | Probe reachability of all ESCs |
| `--probe-sweep` | Sweep all selection values 0-255 |
| `--probe-tries N` | Attempts per probe (default: 6) |
| `--probe-sleep S` | Delay between probe attempts in seconds (default: 0.15) |
| `--settle-ms N` | Delay after entering passthrough in ms (default: 200) |
| `--trace` | Enable detailed 4-way transaction logging |
| `--loglevel LEVEL` | `INFO`, `DEBUG`, or `TRACE` |

### Flash timing

| Flag | Default | Description |
|------|---------|-------------|
| `--flash-inter-esc-ms N` | 250 | Delay between ESCs (ms) |
| `--flash-preselect-tries N` | 2 | Selection attempts before flashing |
| `--flash-preselect-sleep-ms N` | 150 | Delay between selection attempts (ms) |
| `--flash-post-select-ms N` | 200 | Delay after selection (ms) |
| `--flash-erase-retries N` | 3 | Page erase retry count |
| `--flash-erase-inter-page-ms N` | 50 | Delay between page erases (ms) |
| `--flash-write-retries N` | 3 | Write block retry count |
| `--flash-write-inter-block-ms N` | 10 | Delay between write blocks (ms) |
| `--verify-read-retries N` | 3 | Verification read retry count |

### C2 mode

| Flag | Description |
|------|-------------|
| `--c2` | Enable C2 mode |
| `--c2-port PORT` | Serial port for C2 interface (required) |
| `--c2-install uno\|nano` | Upload C2 interface firmware to Arduino |
| `--c2-install-hex PATH` | Explicit path to interface firmware hex |
| `--c2-detect` | Check if C2 interface is present |
| `--c2-read-info` | Read target device ID and revision |
| `--c2-erase` | Erase target MCU |
| `--c2-write-hex PATH` | Write hex file to target |
| `--c2-reset` | Reset target |
| `--c2-timeout-ms N` | C2 command timeout in ms (default: 2000) |
| `--c2-connect-delay-ms N` | Delay after opening the port in ms (default: 2000) |

### Output

| Flag | Description |
|------|-------------|
| `--json` | JSON output for `--list-ports` |
| `--ui-json` | NDJSON event stream used by the GUI to communicate with the CLI |

## Settings

Use `--set KEY=VALUE` with `--write-settings` or when flashing. The option can be repeated to change multiple settings.

| Key | Range | Notes |
|-----|-------|-------|
| `STARTUP_POWER_MIN` | 1000-1125 | Display value (µs) |
| `STARTUP_POWER_MAX` | 1004-1300 | Display value (µs) |
| `COMMUTATION_TIMING` | 1-5 | 1 = Low, 5 = High |
| `DEMAG_COMPENSATION` | 1-3 | 1 = Off, 2 = Low, 3 = High |
| `RPM_POWER_SLOPE` | 0-13 | |
| `BEEP_STRENGTH` | 0-255 | |
| `BEACON_STRENGTH` | 0-255 | |
| `BEACON_DELAY` | 1-5 | |
| `POWER_RATING` | 1-2 | 1 = 1S, 2 = 2S+ |
| `TEMPERATURE_PROTECTION` | 0-7 | |
| `FORCE_EDT_ARM` | On / Off | |
| `BRAKE_ON_STOP` | On / Off | |
| `BRAKING_STRENGTH` | 0-255 | |
| `MOTOR_DIRECTION` | 1-4 | |
| `LED_CONTROL` | 0-4 | |
| `STARTUP_BEEP` | 0-3 | |
| `DITHERING` | On / Off | |
| `PWM_FREQUENCY` | 0 / 24 / 48 / 96 | 0 = Dynamic |

## Building

### Linux

Requires `cmake`, `ninja`, a C++17 compiler, and Qt 6 for the GUI.

On Arch:

```sh
sudo pacman -S cmake ninja gcc qt6-base
```

On Ubuntu/Debian:

```sh
sudo apt install cmake ninja-build g++ qt6-base-dev
```

To build the CLI:

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
./build/prober --list-ports
```

To build and package both the CLI and GUI:

```sh
./scripts/package_linux.sh
```

Output goes to `dist/`:

```
dist/
  prober
  prober_gui
  tools/
    gui/
    c2_firmware/
    bluejay_firmware/
```

Run the GUI:

```sh
./dist/prober_gui
```

### Windows (native)

Requires Visual Studio 2022 or newer, CMake and Ninja, with Qt 6 for the GUI.

To build the CLI:

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release
cmake --build build -j
```

To build both the CLI and GUI:

```sh
cmake -B build -G Ninja -DCMAKE_BUILD_TYPE=Release -DCMAKE_PREFIX_PATH="C:/Qt/6.x.x/msvc2022_64"
cmake --build build -j
```

To build the distribution and installer:

```sh
.\scripts\package_windows.ps1 -QtPath "C:\Qt\6.x.x\msvc2022_64"
```

Or via the `.cmd` wrapper:

```sh
.\scripts\package_windows.cmd -QtPath "C:\Qt\6.x.x\msvc2022_64"
```

### Cross-compiling for Windows on Linux

Requires a `mingw-w64` toolchain, `mingw-w64-qt6-base` (AUR on Arch) and `nsis` for the installer.

On Arch:

```sh
sudo pacman -S mingw-w64-gcc
yay -S mingw-w64-qt6-base nsis
```

Build and package:

```sh
./scripts/package_windows_cross.sh
```

Output goes to `dist/` with the same layout as the native Windows build, including `prober-x.x.x-setup.exe` if NSIS is available.

```
dist/
  prober.exe
  prober_gui.exe
  prober-x.x.x-setup.exe
  tools/
    gui/
    avrdude/
    c2_firmware/
    bluejay_firmware/
```

## Third-party components

The `tools/` directory contains the firmware and external tools shipped with prober. CSerialPort is used when building the application.

| Component | Path | Source | License |
|-----------|------|--------|---------|
| Bluejay firmware | `tools/bluejay_firmware/` | [bird-sanctuary/bluejay](https://github.com/bird-sanctuary/bluejay) | GPL-3.0 |
| avrdude | `tools/avrdude/` | [avrdudes/avrdude](https://github.com/avrdudes/avrdude) | GPL-2.0 |
| C2 firmware | `tools/c2_firmware/` | [stylesuxx/arduino-c2-flasher](https://github.com/stylesuxx/arduino-c2-flasher) | GPL-3.0 |
| CSerialPort | (build dependency) | [itas109/CSerialPort](https://github.com/itas109/CSerialPort) | LGPL-3.0 |

The bundled Bluejay HEX files are unmodified copies from the [official releases](https://github.com/bird-sanctuary/bluejay/releases). You can also use your own firmware with `--hex PATH`.

## License

Project code is licensed under the [GNU AGPLv3](LICENSE). Third-party components retain their respective licenses.
