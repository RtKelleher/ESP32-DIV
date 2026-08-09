# ESP32-DIV V2 hardware evidence

The canonical target is ESP32-DIV V2 / ESP32-S3. This file deliberately distinguishes
facts by evidence source. Nothing in this audit was tested on physical hardware.

## Confirmed by schematic or BOM

- MCU module: ESP32-S3-WROOM-1U-N16.
- Display/controller: 2.8-inch 240x320 ILI9341 and XPT2046 touch controller.
- I/O expander: PCF8574T.
- Storage: microSD wired for SPI.
- Power IC listed in the main BOM: IP5306 in SO-8 package.
- Main board includes four WS2812B-2020 LEDs, a CP2102 USB-UART bridge, and a buzzer.
- Shield BOM lists three NRF24L01 modules and one CC1101 module.

These are design files/BOM claims, not proof of what any clone actually populated.

## Confirmed by source configuration

- Canonical shared storage/radio bus: SCLK 12, MOSI 11, MISO 13.
- microSD CS 10 and card-detect input 38.
- CC1101 CS 5, GDO0 6, and GDO2 3.
- NRF24 CE/CSN pairs: 15/4, 47/48, and 14/21.
- TFT: SCLK 36, MOSI 35, MISO 37, CS 17, DC 16, reset 0.
- Touch: same display signal bus, CS 18; `Touchscreen.cpp` treats IRQ as absent.
- Backlight control: GPIO 7.
- NeoPixel data: GPIO 1.
- IR RX/TX: GPIO 21/14, overlapping NRF24 #3.
- GPS RX/TX: GPIO 5/6, overlapping CC1101 control pins.
- Optional PN532 source: software SPI on clock 12, MOSI 13, MISO 11, CS 5.
- V2 source defines both buzzer and battery ADC pins as `-1` despite schematic nets
  around GPIO 2.

## Confirmed on physical hardware

None during Phase 0/1.

## Assumed or requires verification

- `TODO(HARDWARE_VERIFY)`: card-detect polarity and whether GPIO 38 is reliably wired
  on production and clone boards.
- `TODO(HARDWARE_VERIFY)`: which IP5306 variant is populated and whether usable I2C
  telemetry lines are routed. The BOM part name alone is insufficient evidence.
- `TODO(HARDWARE_VERIFY)`: whether the GPIO 2 divider is a valid, calibrated battery
  voltage signal and whether it can coexist with the buzzer circuit.
- `TODO(HARDWARE_VERIFY)`: actual flash/PSRAM manufacturer behavior under the declared
  16 MB flash and OPI PSRAM build settings.
- `TODO(HARDWARE_VERIFY)`: PCF8574 address; source scans 0x20 through 0x27.
- `TODO(HARDWARE_VERIFY)`: safe optional configurations for GPS and PN532 given the
  source-level pin conflicts.

## Power audit finding

Current V2 source sets the battery ADC pin to `-1`, but `readBatteryVoltage()` still
passes that value to ADC setup/read functions and maps the result to a percentage.
The status bar can therefore display unsupported or bogus telemetry. A later power
service must separately describe charging state, power path, button behavior, measured
voltage, and percentage estimation. Unsupported values must be reported as unavailable.

## Capability-probing constraints

Safe boot probing should be one-time or explicitly cached, should never select two
shared-SPI clients, and should avoid destructive RF activity. Pin-conflicting optional
devices cannot all be probed blindly. The eventual `BoardCapabilities` result should
record `present`, `absent`, `not configured`, and `probe failed` distinctly.
