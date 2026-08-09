# ESP32-DIV V2 experiment log

This is the laboratory notebook for the assembled reference unit. A result is not
`CONFIRMED_RUNTIME` until it appears here with enough detail to reproduce it.

## Status vocabulary

`PASS` · `FAIL` · `INCONCLUSIVE` · `NOT RUN`

## Experiment template

```text
Experiment ID:
Date/time/timezone:
Operator:
Firmware commit:
Build environment:
Hardware unit/revision:
Power source and battery state:
Objective:
Setup:
Tools/media/test devices:
Steps:
Expected result:
Observed result:
Result: PASS | FAIL | INCONCLUSIVE
Relevant serial output:
Artifacts/photos/files:
Safety constraints:
Follow-up:
Related investigation:
```

Never overwrite an old result. Add a new run suffix such as `SD-001-R2` when repeating
an experiment after changing one controlled variable.

## Planned initial experiments

| ID | Objective | Status | Related investigation |
| --- | --- | --- | --- |
| HW-001 | query chip, revision, flash, PSRAM, heaps, partition, reset reason | NOT RUN | HW-001 |
| HW-002 | verify independent SPI communication with all three nRF modules | NOT RUN | HW-002 |
| HW-003 | compare stable, non-destructive nRF register behavior | NOT RUN | HW-003 |
| HW-004 | read CC1101 identity/version/status without transmitting | NOT RUN | HW-004 |
| HW-005 | validate shared-SPI CS/CE idle states and transitions | NOT RUN | HW-005 |
| HW-006 | scan I2C and investigate possible IP5306 response safely | NOT RUN | HW-006 |
| HW-007 | determine PCF8574 address and physical button mapping | NOT RUN | HW-007 |
| HW-008 | characterize both USB connectors and recovery behavior | NOT RUN | HW-008 |
| SD-001 | reproduce the 64 GB mount failure without changing firmware | NOT RUN | SD-001 |
| SD-002 | compare the same card as FAT32 and exFAT, plus known-good FAT32 | NOT RUN | SD-002 |
| SD-003 | compare SD access before/after CC1101 and each nRF transition | NOT RUN | SD-003 |
| SD-004 | exercise boot mount, retry, removal, reinsertion, and remount | NOT RUN | SD-004 |
| SD-005 | record raw card-detect state with/without media | NOT RUN | SD-005 |
| PWR-001 | measure 3.3 V rail at idle | NOT RUN | PWR-001 |
| PWR-002 | measure rail behavior during existing RF loads | NOT RUN | PWR-002 |
| PWR-003 | characterize charging and power-button behavior | NOT RUN | PWR-003 |
| RF-001 | compare passive/register/owned-receive behavior of three nRF modules | NOT RUN | RF-001 |
| RF-002 | characterize CC1101 passive receive against an owned source | NOT RUN | RF-002 |

## Results

No controlled experiments have been recorded yet.
