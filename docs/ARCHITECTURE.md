# Phase 0/1 architecture audit

Audit baseline: `9d4d82fe7a12febf554b12e1eca6d434ebe79d39`.
Scope: intake and audit only. No firmware architecture or behavior was changed.

## Executive assessment

The firmware contains substantial working hardware- and protocol-specific code worth
preserving. Its principal reliability problem is not a lack of features; it is the
absence of explicit ownership and lifecycle boundaries around those features. The
current sketch combines UI dispatch, mutable application state, peripheral setup,
storage recovery, background tasks, and feature control in a few very large translation
units. The V2 source also declares mutually conflicting pin uses.

The smallest safe path is to freeze the build, establish one V2 hardware definition,
centralize shared-SPI/storage ownership, and only then migrate blocking features one at
a time into a central enter/tick/exit loop. A wholesale rewrite would carry an
unacceptably high regression risk.

## Repository and source inventory

The main firmware directory contains approximately 47,900 lines. Largest units:

| File | Approximate lines | Primary responsibility |
| --- | ---: | --- |
| `wifi.cpp` | 10,669 | Wi-Fi tools, scanning, captures, captive/OTA paths |
| `bluetooth.cpp` | 9,566 | BLE and NRF24/ESB-related tools |
| `ESP32-DIV.ino` | 4,587 | boot, menu arrays, dispatch, top-level state |
| `subghz.cpp` | 4,279 | CC1101/sub-GHz features |
| `gps.cpp` | 4,035 | GPS tools and wardriving tasks |
| `utils.cpp` | 3,156 | UI/system helpers, SD recovery, status task |
| `ir.cpp` | 3,151 | infrared features |
| `rfid.cpp` | 2,632 | optional PN532 features |
| `ducky.cpp` | 1,476 | script execution and file access |
| `shared.h` | 825 | broad includes, pins, globals, cross-module contracts |

Smaller existing seams worth preserving and strengthening include `SettingsStore`,
`KeyboardUI`, `Touchscreen`, and `Theme`.

Other tracked content includes 130 graphics assets, patched/supplied libraries,
V1/V2 Gerbers and schematics/BOMs, precompiled firmware, a browser flasher, a Python
desktop flasher, 3D models, and Ducky scripts. There is no PlatformIO configuration and
no CI workflow that compiles the firmware.

## Current control flow

`setup()` initializes display/input, storage/settings, menu state, and background
services. The Arduino `loop()` calls theme application, button handling, and status-bar
updates. `handleButtons()` switches on menu/submenu indexes. Submenu handlers then call
a feature-specific `Setup()` and enter a feature-owned `while` loop that repeatedly
calls that feature's `Loop()` until global exit conditions change.

Consequences:

- the apparent central loop is suspended while a feature runs;
- features vary in exit behavior and often contain additional blocking loops;
- input release waits and scans add nested blocking intervals;
- application state is encoded indirectly by menu indexes and shared booleans;
- status, storage recovery, task pauses, and teardown cannot be enforced uniformly.

The namespaces and `Setup`/`Loop` pairs provide useful migration seams, but they are
not yet a common lifecycle. The first lifecycle migration should retain each feature's
implementation and adapt only its scheduling and resource acquisition.

## Global-state analysis

The sketch holds global current menu/submenu indexes, page indexes, arrays of names and
icons, per-menu page counts, initialization flags, layer flags, and active submenu
arrays. Other modules share `tft`, `pcf`, `UI`, `feature_active`,
`feature_exit_requested`, `in_sub_menu`, `is_main_menu`, and related flags through
broad headers and `extern` declarations.

Each large feature file also owns extensive namespace/file-static state: scan results,
mode flags, timers, buffers, `String` arrays, vectors, task handles, and device objects.
There are hundreds of `String`, vector, allocation, and free sites. This does not prove
fragmentation, but it creates a measurement priority for long-running scans, UI logs,
capture lists, and repeated feature entry/exit.

Duplicated state is especially dangerous in storage: utilities, settings, Ducky,
Wi-Fi PCAP/captive paths, and sub-GHz code maintain separate mounted/available flags.
No one value is authoritative.

## SD lifecycle analysis

Boot initializes SD before loading settings. V2 utility code uses card-detect GPIO 38,
tries `SD.begin()` at 4 MHz, caches a mounted flag, and uses `SD.cardType()` as a later
liveness check. Its recovery path can deselect devices, call `SD.end()`/`SPI.end()`,
reset shared pins, restart SPI, and remount.

Problems:

- mount ownership is duplicated and some features call `SD.begin()` directly;
- recovery can invalidate open `File` objects held by other code or the GPS task;
- no bus/file lock prevents concurrent recovery;
- failures collapse to booleans or serial strings and do not expose an error reason;
- mount attempts, actual clock, capacity, card type, and filesystem are not retained;
- active-feature checks are an incomplete substitute for ownership;
- successful post-radio restoration is assumed rather than centrally verified.

ESP32 Arduino core 2.0.10 is built with FatFs exFAT support disabled
(`FF_FS_EXFAT == 0`). The current `SD` path therefore cannot claim generic exFAT or
arbitrary 64 GB-card support. A known-good FAT32 card is needed for hardware testing.
SdFat/exFAT evaluation belongs in Phase 5 and must include shared-bus correctness,
firmware update/capture compatibility, and measured flash/RAM cost before migration.

## FreeRTOS and background-task analysis

| Task | Core/stack/priority | Work | Main risk |
| --- | --- | --- | --- |
| Wi-Fi background scanner | core 0 / 4096 / 1 | periodic scan/cache | shared flags/results use partial synchronization |
| BLE background scanner | core 0 / 4096 / 1 | blocking two-second scan/cache | foreground/background result ownership |
| status-bar task | core 0 / 2048 / 1 | battery/SD/radio counters | reads invalid V2 battery pin; task creation unchecked |
| GPS foreground wardrive | core 0 / 10240 / 1 | scan and SD writes | receives pointer to stack-local `File`; long stop wait |
| GPS background wardrive | core 0 / 10240 / 1 | scans and owns SD file | stop timeout can clear control while task remains active |

Several shared control variables are `volatile`, which does not make multi-core access
atomic or provide a consistency protocol. Small result caches use critical sections in
places, but there is no storage/SPI owner shared with tasks. Some task-creation results
are ignored. Task handles are changed by workers and controllers without a single
lifecycle invariant.

## Top 15 architectural and reliability problems

| Rank | Severity | Problem | Why it matters |
| ---: | --- | --- | --- |
| 1 | Critical | Three compiler-detected Wi-Fi frame array overflows | Immediate memory-corruption risk in existing code |
| 2 | Critical | No shared-SPI owner or transaction discipline | Devices/tasks can corrupt one another's configuration and selection |
| 3 | Critical | V2 pin conflicts across IR/NRF3, GPS/CC1101, PN532/CC1101 | Some advertised configurations are electrically/software mutually exclusive |
| 4 | Critical | SD can be ended/remounted while another task owns files | Data loss, invalid objects, hangs, and watchdog risk |
| 5 | High | Feature-owned blocking application loops | Prevents uniform input, status, teardown, and failure handling |
| 6 | High | Manual, globally mutated, incompletely pinned build | Baselines cannot be reliably reproduced or reviewed |
| 7 | High | V2 battery pin `-1` is still read and converted to percentage | Bogus telemetry and invalid GPIO API use |
| 8 | High | Multiple independent SD mount flags and direct mount calls | No single source of truth or deterministic recovery |
| 9 | High | Background-task lifecycle relies on volatile flags/timeouts | Races can outlive feature state and local objects |
| 10 | High | Giant translation units and colliding macros | Hidden coupling; warnings already show redefinitions |
| 11 | High | Menu metadata duplicated across arrays, indexes, handlers | Adding/reordering features can silently desynchronize UI and behavior |
| 12 | High | No general capability model/probe result | Missing or clone-variant hardware can present misleading features or fail badly |
| 13 | Medium | Boolean/silent subsystem errors and weak diagnostics | Field failures cannot be distinguished or acted on safely |
| 14 | Medium | Unmeasured dynamic allocation and `String` churn | Long-running stability risk, especially around scans and logs |
| 15 | Medium | Firmware already consumes 88% of application partition | Small dependency/diagnostic growth can break release builds |

The array overflows are concrete compiler findings, not architectural speculation.
They should be fixed as a narrowly reviewed reliability patch before broad lifecycle
work, even though Phase 0/1 intentionally leaves source unchanged.

## Recommended migration order

1. **Phase 2 build:** encode the exact core/library/tool assumptions in PlatformIO,
   reproduce size/behavior, add compile CI, and expose project warnings.
2. **Immediate bounded safety fixes:** repair the three array bounds issues with focused
   tests/review; do not mix them with architecture moves.
3. **Canonical V2 board facts:** consolidate pins, mutually exclusive options, and
   evidence annotations without changing feature behavior.
4. **Shared SPI ownership:** add a minimal exclusive session model and migrate SD,
   CC1101, then NRF24 clients individually. Treat PN532 as an explicit optional mode.
5. **Storage service:** make mount state authoritative, synchronize open files/remount,
   add typed errors and diagnostics, then remove direct mount paths.
6. **Capability snapshot:** perform safe, cached boot probes through the resource owners.
7. **Application state/lifecycle:** create a central active-feature state and adapt one
   low-risk feature at a time from blocking loops to enter/tick/exit.
8. **Feature registry/menu:** generate menu entries only after lifecycle metadata is
   reliable; remove old arrays and switch cases in the same migration.
9. **Task ownership:** attach tasks to services/features with deterministic cancel/join
   semantics and no borrowed stack-local resources.
10. **Diagnostics/tests:** expose build, resources, storage, peripherals, and power truth;
    expand native and hardware smoke tests.
11. **Cleanup:** delete replaced globals/APIs, tighten warnings, and measure allocations,
    stacks, heap floor, and binary-size changes.

## Regression-risk map

| Area | Risk | Why | Required evidence before cutover |
| --- | --- | --- | --- |
| shared SPI and SD | Critical | cross-feature recovery and open files | bus invariant tests plus V2 FAT32 hardware matrix |
| Wi-Fi frame tools | Critical | existing bounds defects; timing-sensitive frames | compile/static checks and controlled existing-behavior test |
| application dispatch | High | every feature entry/exit depends on it | lifecycle transition tests and per-feature smoke checklist |
| GPS tasks/storage | High | multi-core scans and long-lived files | cancellation/join tests and sustained SD logging |
| BLE/NimBLE | High | stack/task state shared across modes | repeated enter/exit and background/foreground coexistence |
| OTA/firmware update | High | flash partition and SD/network file paths | signed/known image update and failure/recovery tests |
| settings | High | boot order and persistent schema | migration/default/corruption host tests |
| CC1101/NRF24 | High | shared pins, modes, frequencies | passive presence tests and feature-by-feature hardware smoke |
| touch/display/menu | Medium | central navigation and patched TFT setup | operator UI checklist plus registry/pagination tests |
| IR/RFID/optional GPS | Medium | conflicting optional pins | explicit configuration and unavailable-state tests |
| file serialization/captures | Medium | behavior spread among features | golden-file/path tests and old-file compatibility |
| docs/assets/flasher UI | Low | little runtime coupling | link/manifest checks and browser smoke test |

## Phase acceptance criteria

Phase 0/1 is accepted when the exact upstream commit and firmware version are recorded,
the unmodified canonical V2 build succeeds in isolation, size and warning findings are
captured, supplied dependencies are inventoried, a baseline tag exists, hardware claims
are evidence-qualified, and no firmware behavior has changed.

## Assessment of the proposed target architecture

The direction is appropriate, with four ESP32-S3-specific cautions:

1. **Do not make every concept a virtual class.** A fixed registry of metadata and
   function pointers or small static adapters may be cheaper and easier to place than
   pervasive heap-allocated polymorphism. The semantic enter/tick/exit contract matters
   more than inheritance.
2. **Avoid one wrapper per hardware noun.** Board facts, shared-bus/storage ownership,
   and genuinely testable services justify boundaries. Thin wrappers around every GPIO
   or library call would increase flash and indirection without improving safety.
3. **Do not use RAII alone as cross-task locking.** Scoped sessions are useful, but the
   owner must define bounded acquisition, FreeRTOS-aware synchronization, task context,
   error results, and recovery when a client fails to release cleanly.
4. **Keep capability probing configuration-aware.** Probing all possible devices at boot
   can itself violate pin exclusivity or disturb a bus. Model `not configured` separately
   and cache safe probe results.

The suggested directory hierarchy is a destination, not a Phase 3 prerequisite. Start
with a few cheap seams at hardware ownership and application scheduling, then move files
only when their dependencies have actually narrowed. A desktop-style Clean Architecture
or domain framework would be inappropriate for this firmware; ports/adapters are useful
only at shared hardware and test seams.

## Remaining technical debt after Phase 1

All ranked issues remain intentionally unfixed. Physical V2 verification, exact original
dependency reconstruction, exFAT/SdFat evaluation, runtime heap/stack measurements, and
feature-by-feature smoke results are outstanding. They belong to later approved phases.
