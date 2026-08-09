# Reference ESP32-DIV V2

This is the canonical hardware knowledge-base entry for the assembled ESP32-DIV V2
currently in hand. The objective is to understand and improve software around this
unit, not redesign it.

Audit source baseline: `9d4d82fe7a12febf554b12e1eca6d434ebe79d39`  
Fork documentation commit at audit start: `54bf20d24af14c53bc261947c57ae7478c4e38c6`  
Firmware version: `v1.7.2`

The upstream MIT license and attribution remain unchanged in `LICENSE`.

## Evidence vocabulary

| Evidence | Meaning |
| --- | --- |
| `CONFIRMED_PHYSICAL` | Marking, connection, or behavior directly observed on the reference unit |
| `CONFIRMED_SCHEMATIC` | Shown in the supplied V2 schematic/BOM |
| `CONFIRMED_SOURCE` | Present in the audited firmware or tracked build configuration |
| `CONFIRMED_RUNTIME` | Demonstrated by a recorded experiment on the reference unit |
| `INFERRED` | Reasonable interpretation that still requires verification |
| `COMMUNITY_REPORT` | Third-party report, not direct evidence for this unit |
| `UNKNOWN` | Insufficient evidence |

Physical markings establish what is printed on a package, not semiconductor
authenticity or complete electrical capability.

## Reference identity

| Item | Observation | Evidence | Confidence | Runtime tested |
| --- | --- | --- | --- | --- |
| MCU module | `ESPRESSIF ESP32-S3-WROOM-1U N16R8` | `CONFIRMED_PHYSICAL` | High | No |
| Flash interpretation | 16 MB | `INFERRED` from N16R8 marking; build assumes 16 MB | High | No |
| PSRAM interpretation | 8 MB | `INFERRED` from N16R8 marking; build enables OPI PSRAM | High | No |
| MCU antenna | external-antenna/U.FL module variant | `CONFIRMED_PHYSICAL` marking and observed antenna path | High | No |
| USB-UART | Silicon Labs CP2102N-family marking | `CONFIRMED_PHYSICAL` | High family-level, exact suffix unknown | No |
| GPIO expander | NXP `PCF8574T` | `CONFIRMED_PHYSICAL` | High | No formal mapping test |
| Power controller | `IP5306` top marking | `CONFIRMED_PHYSICAL` | High marking, low variant capability | No |
| Touch controller | XPT2046-family | `CONFIRMED_PHYSICAL`, `CONFIRMED_SCHEMATIC` | High family-level | Device works; no recorded diagnostic |
| 2.4 GHz modules | three modules with nRF24L01+-class radio, RFX2401C-class front end, 16 MHz crystal, and U.FL | `CONFIRMED_PHYSICAL`; three nRF24 modules also `CONFIRMED_SCHEMATIC` | Medium identity, high count/topology | No formal register test |
| Sub-GHz module | red `AS07-M1101S V3.1`, CC1101-class | `CONFIRMED_PHYSICAL`; CC1101 also `CONFIRMED_SCHEMATIC` | Medium identity | No formal register test |
| microSD | onboard socket | `CONFIRMED_PHYSICAL`, `CONFIRMED_SCHEMATIC` | High | Reported 64 GB failure; controlled reproduction pending |

The machine-readable form is `hardware/reference_v2.yaml`.

## Source/module inventory

The firmware is approximately 47,900 lines. Its principal modules are:

| Module | Approx. lines | Hardware or application role |
| --- | ---: | --- |
| `ESP32-DIV.ino` | 4,587 | boot, menu/index dispatch, top-level globals |
| `wifi.cpp` | 10,669 | ESP32 Wi-Fi, captures, SD/OTA paths, background scan |
| `bluetooth.cpp` | 9,566 | NimBLE plus all nRF24 modes and raw SPI radio access |
| `subghz.cpp` | 4,279 | CC1101 modes, EEPROM profiles, SD exports/logging |
| `gps.cpp` | 4,035 | optional GPS UART, Wi-Fi/BLE scans, SD logging, tasks |
| `utils.cpp` | 3,156 | SD recovery, I2C/PCF init, battery/status, system UI |
| `ir.cpp` | 3,151 | IR receive/transmit and SD profiles |
| `rfid.cpp` | 2,632 | optional PN532 software-SPI path |
| `ducky.cpp` | 1,476 | BLE HID scripting and SD files |
| `shared.h` | 825 | board variants, pins, feature flags, shared declarations |

Existing narrower seams include `SettingsStore`, `Touchscreen`, `KeyboardUI`, and
`Theme`. Detailed Phase 1 application analysis remains in `docs/ARCHITECTURE.md`.

## Build and dependency inventory

The unmodified baseline compiles with ESP32 Arduino core 2.0.10, a manually replaced
`platform.txt`, a supplied/patched TFT_eSPI 2.5.43 ZIP, a supplied CC1101 2.5.7 ZIP,
and globally sourced Arduino libraries. The tracked map confirms NimBLE-Arduino 1.4.2
and an ArduinoJson 6.18.0 ABI; several other exact original library versions are not
recoverable from the repository alone.

Baseline build:

```text
Sketch: 1,749,225 / 1,966,080 bytes (88%)
Static/global RAM: 135,080 / 327,680 bytes (41%)
```

See `docs/BUILD.md` for the command, dependency evidence, warning audit, hashes, and
proposed PlatformIO structure.

## FreeRTOS task inventory

| Task | Core | Stack | Priority | Lifecycle/ownership finding |
| --- | ---: | ---: | ---: | --- |
| `bgWifiScan` | 0 | 4096 | 1 | permanent task; async scan controlled by volatile flags and menu globals |
| `bgBleScan` | 0 | 4096 | 1 | permanent task; performs blocking two-second scans and shares `bleResults` |
| `statusBar` | 0 | 2048 | 1 | permanent task; samples SD/battery/radio counters every 400 ms |
| `wardfg` | 0 | 10240 | 1 | feature task; receives a pointer to a caller-owned `File` object |
| `wardrive` | 0 | 10240 | 1 | background GPS/radio/SD logger; owns an open file until stopped |

No code records task stack high-water marks. Several task creation results are ignored.
`volatile` flags are used as cross-core coordination but do not provide an ownership
protocol. The GPS stop paths use bounded polling and can return before task termination.

## Global-state inventory

State is distributed among:

- application navigation: menu/submenu indexes, pages, layers, initialization flags;
- lifecycle flags: `feature_active`, `feature_exit_requested`, `in_sub_menu`, and
  `is_main_menu`;
- global devices: TFT, PCF8574, touchscreen SPI/controller, feature radio objects;
- feature-local globals: scan results, mode flags, buffers, timers, files, and vectors;
- background state: task handles, scan-running flags, cached counts/results;
- duplicated storage state: core SD mount state plus settings, Wi-Fi PCAP/captive,
  Ducky, sub-GHz, and detector-specific flags;
- UI state: status bar, touch navigation, theme, and large per-feature log buffers.

There is no single authoritative application state, SPI owner, storage owner, or radio
capability snapshot.

## Memory and PSRAM assumptions

- The physical N16R8 marking strongly indicates 8 MB PSRAM, and the baseline Arduino
  board options select OPI PSRAM.
- Firmware never calls `psramFound()`, reports PSRAM size/free space, or records the
  runtime flash/partition layout.
- No deliberate PSRAM allocation (`ps_malloc`, `MALLOC_CAP_SPIRAM`, or equivalent) is
  present. Large static arrays remain ordinary static storage; ordinary `malloc`/`new`
  placement is not measured.
- V2 compile-time constants intentionally allocate larger scan/capture buffers than V1.
- Significant dynamic users include Wi-Fi AP lists, PCAP pools, `String` log arrays,
  vectors, BLE objects/results, and a dynamic `NetworkInfo` array.
- No important task reports stack high-water marks, largest free internal block, minimum
  free heap, or PSRAM fragmentation.

The first memory change should be instrumentation. Do not move allocations to PSRAM
until lifetime, access pattern, internal-DMA requirements, and measured pressure are
known.

## Where V1/CYD support complicates V2

`shared.h` contains more than 80 board-condition branches covering pin maps, touch
rotation, bus selection, buffers, battery ADC, and capabilities. Additional branches
in boot, SD recovery, BLE, Wi-Fi, and touch code change initialization and task behavior.
Three separate TFT_eSPI setup files must be manually selected.

Specific complications include:

- `BOARD_HAS_ESP32S3` is used as a proxy for memory, BLE availability, and boot behavior;
- V1 defers SD/BLE/background tasks because of reported WDT/heap problems;
- CYD and V1 touch use a separate classic-ESP32 VSPI path while V2 uses an HSPI object;
- shared constants contain three pin maps and optional-device defaults;
- V2 fixes can be obscured by compatibility conditionals in the same functions.

Early work should compile V2 canonically and preserve V1/CYD source without spending
substantial effort validating those targets.

## Hardware access points by source file

| File | Direct hardware access |
| --- | --- |
| `ESP32-DIV.ino` | boot serial, TFT, backlight PWM, PCF button init, background task startup |
| `shared.h` / `BoardConfig.h` | board selection, pin facts, optional-feature defaults |
| `Touchscreen.cpp` | XPT2046, SPI instance creation/reinitialization, touch calibration |
| `utils.cpp` | shared SPI recovery, SD begin/end, PCF/I2C, ADC battery read, status task |
| `SettingsStore.cpp` | SD files and directories |
| `wifi.cpp` | ESP Wi-Fi driver, packet callbacks, SD capture/update files, Wi-Fi task |
| `bluetooth.cpp` | NimBLE, RF24 objects, raw nRF24 SPI/register access, SD logging, BLE task |
| `subghz.cpp` | CC1101 library, GDO pins, raw shared-bus transitions, EEPROM, SD files |
| `gps.cpp` | UART2, Wi-Fi/BLE scans, SD files, two tasks |
| `rfid.cpp` | optional PN532 software SPI and shared-bus restoration |
| `ir.cpp` | IR receiver/transmitter and SD profiles |
| `ducky.cpp` | NimBLE HID and SD scripts |

## Top reliability risks

1. Three compiler-detected out-of-bounds Wi-Fi frame writes.
2. No exclusive owner or transaction invariant for shared SPI.
3. SD can be ended/remounted while a task owns an open file.
4. Pin conflicts are not modeled as mutually exclusive configurations.
5. V2 battery code reads GPIO `-1` and derives a percentage from it.
6. TFT/touch share physical signal pins but are represented by separately initialized
   software SPI clients without explicit arbitration.
7. Feature-owned blocking loops prevent uniform teardown and resource restoration.
8. nRF and CC1101 initialization is duplicated across feature modes.
9. Sub-GHz exit/restoration behavior is not uniform across all modes.
10. Background tasks share mutable scan/storage state with foreground features.
11. Build dependencies and patched platform behavior are not reproducibly pinned.
12. Warning suppression hides memory-safety and truncation findings.
13. Large dynamic/static allocations are not measured against internal RAM/PSRAM.
14. Capability absence is often inferred through feature behavior rather than probed.
15. Application partition usage is already 88%, with no size regression gate.

## Top observability gaps

- runtime chip revision, flash size, PSRAM size/free, partition table, and reset reason;
- free internal heap, minimum heap, largest block, and task stack high-water marks;
- current SPI owner, configuration, selected CS lines, and transition history;
- SD filesystem, card capacity/type, attempt count, actual speed, and structured error;
- complete I2C scan and explicit PCF8574/IP5306 results;
- independent nRF register sanity and presence for all three modules;
- CC1101 part/version register and configured state;
- touch/display bus state and failed sample counts;
- real battery measurement source, charge state, and unsupported telemetry state;
- native USB versus CP2102N port identity and active console path.

## Evidence discrepancies

| Subject | Physical | Schematic/BOM | Firmware/documentation | Status |
| --- | --- | --- | --- | --- |
| ESP module | N16R8 | module/BOM says N16 | build selects 16 MB + OPI PSRAM but reports neither | runtime verification required |
| USB-UART | CP2102N-family | CP2102 | generic USB description | exact suffix and port mapping unknown |
| IP5306 | top marking only | LED1/LED2 pins share `SCL`/`SDA` nets | no IP5306 probe or driver | capability unknown |
| battery/buzzer | circuits physically not traced | both meet at GPIO2-related circuitry | both pins set `-1`, but ADC code still executes | unsafe/unsupported software state |
| TFT reset | not traced | TFT reset connects to board `RESET` net | TFT_eSPI setup defines GPIO0 | conflict requires measurement/trace |
| SD detect | slot present | socket pin labeled NC is routed to GPIO38 | active-low input-pullup assumed | polarity/contact behavior unknown |
| SD chip-select symbols | socket CS is GPIO10 | CS is GPIO10 | canonical `SD_CS` is 10, but legacy `SD_CS_PIN` defaults to 5 and must be filtered against CC1101 | duplicate definition is hazardous |
| nRF scanner bus | three populated modules use common 12/11/13 bus | common 12/11/13 bus | actual defines match SD, but an exit comment claims scanner SCK/MISO are swapped | source comment is stale; PN532 is the swapped client |
| NeoPixels | not formally exercised | four WS2812 devices on GPIO1 | setting changes a boolean only | documented feature is not implemented in hardware control |
| optional PN532 | not observed | absent from base V2 schematic/BOM | software-SPI pins conflict with CC1101 and swap shared data roles | optional external configuration only |
| optional GPS | not observed | absent from base V2 schematic/BOM | GPIO5/6 overlap CC1101 CS/GDO0 | optional external configuration only |

## Intake conclusion

The supplied design files are useful topology evidence, but the assembled unit—not a
hypothetical BOM—is authoritative for experiments. Physical observations upgrade the
reference target to N16R8. Runtime facts remain unproven until recorded in
`docs/EXPERIMENT_LOG.md`.
