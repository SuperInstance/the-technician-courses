# A3 — OpenConstruct ESP32 Build (Worked Exercise)

Serves: **Phase 2 — Assistant** (per `06-EDUCATIONAL-PLATFORM.md`), a home exercise reviewed by the trainer in minutes, not hours.
Source repo: [`SuperInstance/openconstruct-esp32`](https://github.com/SuperInstance/openconstruct-esp32), branch `production-round3-2026-07-10`.

This exercise walks through building the OpenConstruct ESP32 firmware from source. It ends with a real, physical object: a flashed sensor node.

## Prerequisites

- [PlatformIO Core](https://platformio.org/install/cli) (`pip install platformio`)
- Git
- (For the flash step only) a real ESP32 dev board and USB cable

## ✅ Clone & Build

This section was actually run in a real sandbox for this exercise — the transcript below is genuine, not simulated.

```
$ git clone https://github.com/SuperInstance/openconstruct-esp32.git
$ cd openconstruct-esp32
$ git checkout production-round3-2026-07-10
$ pio run -e esp32dev
```

Real output:

```
Processing esp32dev (platform: espressif32; board: esp32dev; framework: arduino)
--------------------------------------------------------------------------------
PLATFORM: Espressif 32 (7.0.1) > Espressif ESP32 Dev Module
HARDWARE: ESP32 240MHz, 320KB RAM, 4MB Flash
PACKAGES:
 - framework-arduinoespressif32 @ 3.20017.241212+sha.dcc1105b
 - tool-esptoolpy @ 2.41100.0 (4.11.0)
 - toolchain-xtensa-esp32 @ 8.4.0+2021r2-patch5
Converting basic_shell.ino
Found 40 compatible libraries
Dependency Graph
|-- PubSubClient @ 2.8.0
|-- src
Building in release mode
Compiling .pio/build/esp32dev/src/basic_shell.ino.cpp.o
Retrieving maximum program size .pio/build/esp32dev/firmware.elf
Checking size .pio/build/esp32dev/firmware.elf
RAM:   [=         ]  13.9% (used 45592 bytes from 327680 bytes)
Flash: [======    ]  58.5% (used 766365 bytes from 1310720 bytes)
========================= [SUCCESS] Took 3.08 seconds =========================

Environment    Status    Duration
-------------  --------  ------------
esp32dev       SUCCESS   00:00:03.082
```

A trainee whose build differs from this — different memory percentages are fine (they mean nothing changed structurally), but a `FAILED` status or a different error means they're debugging their own environment, not this content. That debugging is itself part of the lesson: read the actual PlatformIO error, don't just re-run the command hoping it changes.

## ⚠️ Flash

Grounded in `platformio.ini` and the PlatformIO docs, but **not executed here** — this sandbox has no attached ESP32 hardware.

```
$ pio run -e esp32dev -t upload
$ pio device monitor
```

`platformio.ini`'s `[env:esp32dev]` targets the generic `esp32dev` board over the default serial upload protocol — no special flags needed for most USB-to-UART ESP32 dev boards. If `pio run -t upload` can't find the port, see Failure Mode 1 below.

## ⚠️ Sensor Reading

Grounded in reading `examples/basic_shell/basic_shell.ino` and `src/` — not executed here (needs the physical board from the Flash step). The example firmware is a shell over the sensor registration API described in the repo's README (`registerSensor(pin, name, type)` / `update()`); after flashing, connecting a serial monitor at the baud rate set in `basic_shell.ino` should show sensor readings printed as they're polled by `update()`.

## Three Likely Failure Modes

1. **⚠️ Serial port not found on upload.** PlatformIO auto-detects the USB serial port; on Linux this is usually `/dev/ttyUSB0` or `/dev/ttyACM0`. If auto-detect fails, pass it explicitly: `pio run -t upload --upload-port /dev/ttyUSB0`. Also check the user is in the `dialout` group (`sudo usermod -aG dialout $USER`, then re-login) — a permissions error here is easy to misread as a hardware problem.
2. **✅ Missing `platformio.ini` or wrong working directory.** This was a real bug found and fixed in an earlier production-hardening round on this exact repo — `platformio.ini` was gitignored and missing from a clean clone. Confirmed fixed on the `production-round3-2026-07-10` branch used in this exercise (the build above succeeded from a fresh clone of that branch). If a trainee sees `platformio.ini not found`, they're likely on an older branch or a stale clone — re-clone from `production-round3-2026-07-10`.
3. **⚠️ Library dependency resolution failure.** The build depends on `PubSubClient @ 2.8.0` (declared in `platformio.ini`). If PlatformIO's library registry is unreachable (offline, firewalled), the build fails at the "Scanning dependencies" step rather than compilation. Fix: run once with connectivity to let PlatformIO cache the dependency locally, or vendor it via `lib_extra_dirs` (already configured in this repo's `platformio.ini`) if working fully offline.
