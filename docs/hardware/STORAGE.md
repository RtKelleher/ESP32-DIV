# Storage knowledge and 64 GB investigation

## Current topology

The microSD socket is on the shared radio/storage SPI bus: SCLK 12, MOSI 11, MISO 13,
CS 10, and an assumed active-low card-detect input on GPIO38. The schematic routes a
socket pin labeled NC to GPIO38; actual contact behavior is not yet verified.

## Current mount lifecycle

1. V2 boot configures card detect and calls `restoreSdAfterSharedSpi()`.
2. Recovery raises known radio/optional CS lines, ends SD and SPI, resets shared pins,
   begins SPI on the shared wiring at 4 MHz, and attempts `SD.begin()`.
3. A private `s_sdFsMounted` flag and last-good CS are cached.
4. `isSDCardAvailable()` checks card detect, then `SD.cardType()` as a liveness test.
5. A dead mount triggers `SD.end()` and a soft remount. A full destructive reclaim is
   attempted only when `feature_active` is false.
6. Radio/RFID exits often call `restoreSdAfterSharedSpi()`.
7. Many feature modules keep their own mounted flags; ESB includes direct `SD.begin()`
   calls that bypass the utility state.

## Unmount and open-file behavior

There is no application-wide open-file registry or lock. `SD.end()` can occur while the
GPS background task owns a `File`, and storage recovery is not synchronized with other
feature file users. A boolean `feature_active` check does not account for background
tasks. This is a concrete lifecycle hazard independent of the 64 GB failure.

## Filesystem support

The baseline ESP32 Arduino 2.0.10 FatFs configuration has `FF_FS_EXFAT` disabled.
Therefore:

- FAT32 is the known filesystem path to test first;
- an exFAT-formatted 64 GB card is expected not to mount through the current library;
- capacity alone does not determine support;
- the current UI cannot distinguish unsupported filesystem from bus/card failure.

This does not prove the reference 64 GB failure is only exFAT. Shared-bus state, CS,
card detect, initialization clock, media behavior, and hardware remain candidates.

## Current observability

Available today:

- card-detect-derived icon/state;
- serial success/failure messages in limited paths;
- boolean availability and `cardType()` liveness checks.

Missing:

- filesystem identification or explicit exFAT-unsupported result;
- card type/capacity in diagnostics;
- actual SPI speed and current bus owner;
- structured last error and attempt counter;
- mount/remount duration and timeout reason;
- open-file count and task owner;
- before/after-radio transition log.

## Experiment order before any storage refactor

| ID | Controlled comparison | Primary question |
| --- | --- | --- |
| SD-001 | current 64 GB card, current format | can the failure be reproduced with serial evidence? |
| SD-002 | same 64 GB card FAT32 vs exFAT; known-good 32 GB FAT32 | is failure filesystem- or card-dependent? |
| SD-003 | mount/read before and after CC1101, then each nRF mode | which transition changes bus/mount state? |
| SD-004 | boot mount, feature mount, removal/reinsert, retry | is lifecycle deterministic and bounded? |
| SD-005 | card-detect raw level with no card/inserted card | does GPIO38 represent presence reliably? |

Each run must record card model, capacity, formatter, partition table, filesystem, power
source, firmware commit, serial output, and exact feature sequence. Do not change the SD
library until these controls separate filesystem limitations from bus lifecycle faults.
