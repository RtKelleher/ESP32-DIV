# Investigation tracker

Statuses: `OPEN`, `REPRODUCED`, `INVESTIGATING`, `UNDERSTOOD`, `FIX_READY`, `FIXED`,
`VERIFIED`, `WONT_FIX`.

Every future fix should reference an investigation and the experiment that established
the mechanism.

| ID | Status | Question | Current evidence | Next evidence needed |
| --- | --- | --- | --- | --- |
| HW-001 | OPEN | Does runtime report 16 MB flash and 8 MB PSRAM? | N16R8 physical marking; build flags | runtime system diagnostic |
| HW-002 | OPEN | Can all three nRF modules communicate independently? | physical count and schematic/source pin map | isolated register reads per CSN/CE pair |
| HW-003 | OPEN | Are nRF registers stable and mutually consistent? | no recorded runtime data | repeated non-destructive register snapshots |
| HW-004 | OPEN | What CC1101 identity/version registers are returned? | physical module and schematic/source | passive register test |
| HW-005 | INVESTIGATING | What is the complete shared-SPI topology and idle state? | source/schematic map completed | logic-level/runtime transition trace |
| HW-006 | OPEN | Does the populated IP5306 expose usable I2C telemetry? | marking; schematic shares SCL/SDA; no source driver | read-only I2C scan, variant research, PCB trace |
| HW-007 | OPEN | What PCF8574 address and button mapping are present? | firmware scans 0x20-0x27 | runtime address and button event log |
| HW-008 | OPEN | Which connector is native USB versus CP2102N, and how do they recover? | physical CP2102N-family; two schematic paths | host enumeration/upload matrix |
| HW-009 | INVESTIGATING | Which GPIO conflicts are real on the assembled configuration? | source/schematic conflict table completed | physical trace and runtime ownership tests |
| SD-001 | OPEN | Can the reported 64 GB failure be reproduced under controlled conditions? | user observation only | card identity/filesystem plus serial log |
| SD-002 | OPEN | Is the failure filesystem-dependent? | core exFAT disabled | same-card FAT32/exFAT comparison and 32 GB control |
| SD-003 | OPEN | Do radio transitions break SD access? | source shows direct bus reconfiguration/remount | before/after transition test matrix |
| SD-004 | OPEN | Is SD mount/unmount/retry lifecycle correct? | duplicated mount flags and unsynchronized files | removal/retry/background-file experiments |
| PWR-001 | OPEN | Is the 3.3 V rail stable at idle? | schematic only | DMM measurement with recorded power state |
| PWR-002 | OPEN | Is the rail stable under existing RF load? | no measurements | scoped/DMM/USB-meter load measurements |
| PWR-003 | OPEN | How does charging and power-key behavior work? | IP5306 marking/schematic only | controlled USB/battery state matrix |
| RF-001 | OPEN | How do the three installed nRF modules compare? | physical modules observed | register, passive, and owned receive comparison |
| RF-002 | OPEN | How does the installed CC1101 receive? | physical module observed | known owned transmitter and recorded RSSI/results |

## Known source findings awaiting dedicated issue IDs

- Three compiler-detected Wi-Fi frame array overflows should receive a narrow reliability
  investigation before correction.
- V2 battery ADC pin `-1` is still read and converted into a percentage.
- TFT reset source/schematic mismatch requires a trace/runtime investigation.
- NeoPixel setting has no hardware-control implementation despite schematic devices.
- GPS and PN532 defaults conflict with populated reference resources and must be modeled
  as optional configurations, not probed blindly.
