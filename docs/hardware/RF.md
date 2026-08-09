# RF hardware knowledge

This document characterizes existing hardware only. It does not recommend replacement,
change transmit power, or add radio capabilities.

## 2.4 GHz modules

The reference unit physically contains three separate daughterboards. Observed module
construction includes an nRF24L01+-class transceiver marking, RFX2401C-class PA/LNA
front-end marking, 16.000 MHz crystal, U.FL/IPEX connector, coax, and external SMA path.
These observations are `CONFIRMED_PHYSICAL`; exact die authenticity is `UNKNOWN`.

The supplied shield schematic independently confirms three nRF24L01 design positions,
separate CE/CSN lines, shared SCLK/MOSI/MISO, and one 10 µF local capacitor per module.
It does not show the PA/LNA implementation represented on the assembled daughterboards.

### Current firmware lifecycle

There is no single nRF service. `bluetooth.cpp` contains several independent lifecycles:

- multi-radio RF24 objects for the three CE/CSN pairs;
- a raw-register scanner using nRF #3 pins;
- raw ESB, replay, and mouse-related modes, also using nRF #3;
- direct shared-SPI reinitialization at 10 MHz/mode 0 for raw modes;
- RF24 objects constructed with a 16 MHz SPI speed request;
- feature-specific power-up, configuration, CE control, and power-down;
- feature-specific calls to restore SD after exit.

`radio.begin()` return values are sometimes used to decide whether configuration
continues, but there is no persistent per-radio capability record or diagnostic register
snapshot. Raw modes do not consistently prove identity before writing configuration.
IRQ pins are not connected in the supplied shield schematic and polling is used.

### Safe first characterization

For each radio independently:

1. hold all other CSN lines high and CE lines low;
2. read a small set of non-destructive registers repeatedly;
3. verify stable, sane values and write/read-back only a reversible configuration field;
4. power down and confirm SD can be remounted/read afterward;
5. record module number, antenna port, voltage condition, register data, and failures.

Passive RPD/carrier observations can follow. Receive tests should use another owned test
device and should not alter the project's transmit-power behavior.

## Sub-GHz module

The red module is physically marked `AS07-M1101S V3.1` and appears CC1101-class with
crystal, matching network, U.FL/IPEX, coax, and SMA path. The supplied schematic and BOM
show a CC1101 design position on 3.3 V with CSN 5, SCLK 12, MOSI 11, MISO 13, GDO0 6,
and GDO2 3. Exact silicon identity remains `UNKNOWN` pending register/runtime evidence.

### Current firmware lifecycle

Each sub-GHz feature independently calls combinations of:

- shared-bus reclaim and SD deselection;
- `setSpiPin()` and sometimes `setGDO()`;
- `Init()`;
- modulation, bandwidth, PA, frequency, and RX/TX/idle configuration;
- optional RCSwitch receive/transmit pin setup;
- feature-specific SD transition and restoration.

Initialization and cleanup are duplicated. Some modes explicitly enter idle and restore
SD on exit; others depend on enclosing menu flow or feature-specific paths. There is no
cached chip/version result and no common “radio unavailable” state.

### Safe first characterization

1. deselect SD and all nRF clients;
2. initialize CC1101 without transmitting;
3. read and record part/version/status registers repeatedly;
4. record configured frequency and passive RSSI in a quiet environment;
5. return to idle, restore SD, and verify a known file read.

Later receive testing should use a known owned transmitter. Relative sensitivity and
frequency accuracy require controlled RF equipment to be meaningful.

## Confidence tracking

Do not use module cost or sales channel as evidence of authenticity. Update
`hardware/reference_v2.yaml` with `CONFIRMED_RUNTIME` only when an experiment record
contains reproducible register or receive results.
