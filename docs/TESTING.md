# Testing baseline and proposal

## Phase 0/1 status

- The exact upstream source compiles for the ESP32-S3 V2 settings.
- No upstream host/unit-test suite was found.
- No automated hardware test mode was found.
- No physical hardware was used in this audit.
- No RF transmission test was performed.

The only current automated evidence is compile/link success and static compiler/source
analysis. Hardware claims are not marked tested.

## Proposed host tests

Introduce the `native` PlatformIO environment only after Phase 2 has reproduced the
firmware build. Extract and test logic as it is migrated, beginning with:

1. application state transitions and enter/tick/exit ordering;
2. feature registry filtering, category ordering, and stable IDs;
3. settings serialization, defaults, and migration;
4. path normalization and capture/update filename handling;
5. storage error/result formatting;
6. profile/signal serialization that has no hardware dependency;
7. deterministic menu pagination from feature metadata.

Avoid mocking all of Arduino. Move genuinely hardware-independent logic behind narrow
interfaces and compile it natively.

## Proposed V2 smoke-test mode

All checks must report `PASS`, `FAIL`, `NOT PRESENT`, or `NOT CONFIGURED`. Automated
checks must not transmit disruptive RF traffic.

- firmware/build metadata;
- display color/tile test and backlight control;
- touch targets and physical button events;
- SD detect, mount, card type, capacity, filesystem result, read/write/remount;
- shared-SPI exclusive ownership and known-idle restoration;
- passive response checks for CC1101 and each configured NRF24;
- PCF8574 scan/address and safe GPIO readback where possible;
- NeoPixel visual sequence requiring operator confirmation;
- IR receive and local-loopback test only when fixture-supported;
- GPS serial receive only when configured;
- power/charging telemetry only when electrically supported.

Required storage matrix:

| Medium | Filesystem | Purpose |
| --- | --- | --- |
| known-good <=32 GB card | FAT32 | baseline compatibility |
| known-good 64 GB card | FAT32 | capacity/shared-bus validation |
| known-good 64 GB card | exFAT | explicit supported/unsupported result |
| no card | n/a | card-detect and bounded retry behavior |
| faulty/slow card | applicable | timeout, watchdog, and recovery behavior |

`TODO(HARDWARE_VERIFY)`: run this matrix on an actual V2 and record board revision,
card model, formatter, filesystem, test firmware commit, and observed SPI clock.

## CI proposal

Phase 2 CI should perform a clean `pio run -e div_v2_release`, cache only downloaded
immutable packages, and enforce a firmware-size budget. Add `pio test -e native` as
host tests appear. Debug warnings should apply to project code without turning
third-party/framework warning noise into failures.
