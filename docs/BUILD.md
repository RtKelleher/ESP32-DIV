# Build baseline

## Current upstream process

The documented upstream build uses Arduino IDE 2.x with:

- Espressif ESP32 Arduino core `2.0.10`.
- Board `ESP32S3 Dev Module`.
- 16 MB flash, `Minimal SPIFFS`, OPI PSRAM, CPU 240 MHz, QIO flash.
- Upload speed 921600 and hardware CDC USB mode.
- A manual replacement of the ESP32 core's `platform.txt`.
- Libraries installed into the global Arduino library directory.
- A matching TFT_eSPI `User_Setup.h` copied manually for the selected board.

The contribution guide refers to `Flash File/platform.txt`, but the tracked file is
`Libraries/platform.txt`. It also instructs users to copy all libraries even though
only two library ZIPs are supplied. These ambiguities make the Arduino IDE process
non-reproducible on a clean machine.

## Clean-room baseline compile

The unmodified commit was compiled in an isolated temporary Arduino CLI data and
user directory, rather than using globally installed libraries.

```text
Arduino CLI: 1.5.2-rc.1
ESP32 Arduino core: 2.0.10
Target: esp32:esp32:esp32s3
Flash: 16M
Partition: min_spiffs
PSRAM: opi
CPU: 240 MHz
Flash mode: qio
```

Equivalent baseline command:

```sh
arduino-cli compile \
  --fqbn 'esp32:esp32:esp32s3:UploadSpeed=921600,USBMode=hwcdc,CDCOnBoot=default,CPUFreq=240,FlashMode=qio,FlashSize=16M,PartitionScheme=min_spiffs,PSRAM=opi' \
  --warnings all \
  ESP32-DIV
```

The isolated core's `platform.txt` first had to be replaced with the tracked patched
copy to reproduce upstream's process.

## Baseline result

```text
Build: success
Sketch: 1,749,225 / 1,966,080 bytes (88%)
Global variables: 135,080 / 327,680 bytes (41%)
Static RAM remaining: 192,600 bytes
Generated binary: 1,749,584 bytes
Tracked upstream V2 binary: 1,743,424 bytes
Difference: +6,160 bytes
```

The build is functionally reproducible but not byte-reproducible. The tracked
repository does not pin every dependency version or record the complete original
tool environment, and the rebuilt binary is 6,160 bytes larger.

Useful ELF section sizes from the baseline build:

| Section | Bytes |
| --- | ---: |
| `.dram0.data` | 25,280 |
| `.dram0.bss` | 109,800 |
| `.iram0.text` | 85,863 |
| `.flash.text` | 1,290,043 |
| `.flash.rodata` | 347,012 |

## Dependency inventory

The following versions were either directly identified from tracked metadata/build
artifacts or selected to reconstruct a compatible clean build. “Reconstructed” is
not proof of the exact upstream release environment.

| Dependency | Version/evidence | Status |
| --- | --- | --- |
| ESP32 Arduino core | 2.0.10 | documented and map-confirmed |
| TFT_eSPI | supplied ZIP reports 2.5.43 | vendored patch required |
| SmartRC-CC1101-Driver-Lib | supplied ZIP reports 2.5.7 | supplied ZIP |
| NimBLE-Arduino | 1.4.2 | map-confirmed |
| ArduinoJson | 6.18.0 ABI in map | map-confirmed |
| PCF8574 library | 2.3.7 | reconstructed |
| XPT2046_Touchscreen | 1.4 | reconstructed |
| rc-switch | 2.6.4 | reconstructed |
| RF24 | 1.4.9 | reconstructed |
| arduinoFFT | 1.6.2 | reconstructed |
| IRremoteESP8266 | 2.8.6 | reconstructed |
| Adafruit PN532 | 1.3.4 | reconstructed |
| Adafruit BusIO | 1.17.4 | reconstructed transitive dependency |

Tracked supplied-file SHA-256 values:

```text
TFT_eSPI.zip  f007a151d0ee73c0f99bc4ac089c947bf49a43a6c3fdd0fcdefa2313282f9531
SmartRC-CC1101-Driver-Lib.zip  649ad62731879d0a49fbc4b307cc9c7c5882d87d3a2e467c58c1d9a09d12f41e
platform.txt  89a341acd7ced833c5abe02693440fbe95d330ca3dffbf1a5dbdabd97b2ce75c
```

The patched `platform.txt` globally adds `NFC_INTERFACE_SPI`, linker option
`-zmuldefs`, and warning suppression (`-w`). These changes need to become explicit,
project-scoped PlatformIO options or deliberate patches rather than modifications to
an Arduino installation.

## Warning audit

With the tracked `-w`, the baseline produces no visible warnings. Removing only that
suppression and compiling at the default warning level succeeds at the same size but
reveals macro redefinitions, deprecated ESP event/TCPIP APIs, an unsafe static
capture in an OTA lambda, and deprecated arduinoFFT APIs.

Compiling project code with all warnings currently fails. Important findings include:

- Out-of-bounds writes to index 26 of 26-byte Wi-Fi frame arrays at
  `wifi.cpp` lines 3094, 3102, and 4316.
- Multiple potentially truncated `snprintf` results in Wi-Fi, GPS, Ducky, and RFID.
- Many macro redefinitions caused by monolithic translation units and reused names.
- An always-false brightness range check on an 8-bit value.
- Numerous unused symbols and incomplete aggregate initializers.

Phase 0/1 records these findings but intentionally does not fix them.

## Proposed PlatformIO layout for Phase 2

```text
platformio.ini
boards/
  esp32_div_v2.json              # only if the stock esp32-s3 board cannot express facts
lib/
  patched_tft_espi/              # exact tracked/pinned source and setup
  patched_cc1101/                # only if the supplied ZIP differs upstream
src/                             # initially compile the existing sketch with minimal moves
test/
  host/
scripts/
  build_metadata.py
.github/workflows/
  firmware-build.yml
```

Proposed environments:

- `div_v2_release`: canonical ESP32-S3 V2 build, optimized diagnostics.
- `div_v2_debug`: same pinned platform and dependencies, project warnings enabled,
  debug logging, no framework warning flood.
- `native`: host-only tests with no Arduino hardware dependency.

Phase 2 should pin the Espressif platform release that contains Arduino core 2.0.10,
pin every library to a release or immutable commit, encode the TFT setup locally, and
add a clean `pio run` CI job. It must first reproduce this baseline before any major
framework or library upgrade.
