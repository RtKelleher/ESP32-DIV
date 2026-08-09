# V2 SPI and resource audit

This document records Phase 1 evidence. No SPI manager has been implemented yet.

## Shared SPI resources

| Device | Clock | MOSI | MISO | CS / CE | Current behavior | Evidence |
| --- | ---: | ---: | ---: | --- | --- | --- |
| microSD | 12 | 11 | 13 | CS 10, detect 38 | hardware SPI, mount at 4 MHz | source; schematic |
| CC1101 | 12 | 11 | 13 | CS 5, GDO0 6, GDO2 3 | feature code reinitializes library/bus | source |
| NRF24 #1 | 12 | 11 | 13 | CSN 4, CE 15 | hardware SPI, code selects 10 MHz/mode 0 | source; shield BOM confirms device |
| NRF24 #2 | 12 | 11 | 13 | CSN 48, CE 47 | hardware SPI, code selects 10 MHz/mode 0 | source; shield BOM confirms device |
| NRF24 #3 | 12 | 11 | 13 | CSN 21, CE 14 | hardware SPI, code selects 10 MHz/mode 0 | source; shield BOM confirms device |
| PN532 (optional) | 12 | 13 | 11 | CS 5 | software SPI; source intentionally swaps data pins | source only |

The PN532 entry uses columns according to signal role: its source defines MOSI 13 and
MISO 11, opposite to the shared hardware bus, and uses the same CS pin as CC1101.
This is not a safe concurrently populated configuration.

## Separate display/touch bus

The patched V2 TFT_eSPI setup defines TFT SCLK 36, MOSI 35, MISO 37, CS 17, DC 16,
reset 0, and touch CS 18. `Touchscreen.cpp` creates a separate SPI instance using
the same 36/35/37 signal pins. This bus is separate from the radio/storage bus but
still has more than one client and needs documented transaction discipline.

## Non-SPI pin conflicts

| Resource A | Resource B | Conflict | Evidence |
| --- | --- | --- | --- |
| NRF24 #3 CE/CSN | IR TX/RX | GPIO 14 and 21 | source; comment acknowledges overlap |
| CC1101 CS/GDO0 | GPS RX/TX | GPIO 5 and 6 | source |
| CC1101 CS | optional PN532 CS | GPIO 5 | source |
| buzzer | battery-divider signal | GPIO 2 | schematic; source currently disables buzzer and ADC on V2 |

These conflicts require a board capability/configuration model. They must not be
resolved by enabling every feature at once. Physical routing and populated options
remain `TODO(HARDWARE_VERIFY)`.

## Current ownership behavior

There is no owner, mutex, or project-level SPI transaction API. Project source has no
calls to `SPI.beginTransaction()`/`SPI.endTransaction()` around its raw transfers.
Feature code repeatedly performs combinations of:

- `SPI.begin(...)`, `SPI.end()`, `SPI.setFrequency(...)`, and mode changes;
- direct chip-select writes and raw `SPI.transfer()` operations;
- `SD.begin()` and `SD.end()`;
- CC1101 library initialization and SPI pin assignment;
- destructive bus recovery that resets shared pins.

`utils.cpp` is a partial coordination point for SD. Its recovery path deselects known
devices, ends SD and SPI, resets pins, restarts the bus, and mounts at 4 MHz. It has
no owner token, does not know whether another task holds an open file, and returns
insufficient diagnostic detail. Bluetooth's ESB path and several other features still
bypass it with direct SD/SPI initialization.

## Resource-conflict conclusions

1. Two devices can be selected or reconfigured without a central invariant.
2. A feature can invalidate another feature's bus configuration or open SD file.
3. The set of “known CS pins” is distributed and configuration-dependent.
4. Optional PN532 cannot be modeled as an ordinary peer on the current source pins.
5. Task-level locking will be required because GPS storage can run on core 0 while
   foreground features manipulate the same storage and bus.

## Smallest coherent Phase 3 change

After the reproducible build is established, introduce one immutable V2 pin/capability
definition and a small shared-bus owner. The first API should only initialize the bus,
deselect all configured clients, serialize sessions, apply per-device mode/frequency,
and restore a known idle state. Migrate SD first, then CC1101, then NRF24 clients, with
a compile and smoke-test checkpoint after each client. Do not generalize this into a
large bus framework or rewrite the radio implementations.
