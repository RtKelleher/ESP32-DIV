# ESP32-DIV V2 resource map

This map describes the current reference design and audited firmware. It does not
propose hardware changes. Pin conflicts are ownership/configuration requirements.

## ESP32 GPIO map

| GPIO | Signal/use | Peripheral | Bus/interface | Evidence | Conflict/ownership requirement |
| ---: | --- | --- | --- | --- | --- |
| 0 | boot button; TFT reset in source | ESP32/TFT | GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | schematic ties TFT reset to `RESET`, not clearly GPIO0 |
| 1 | WS2812 data | four NeoPixels | one-wire waveform | `CONFIRMED_SCHEMATIC` | firmware has no hardware driver |
| 2 | battery divider and buzzer transistor control | power/buzzer | ADC/GPIO | `CONFIRMED_SCHEMATIC` | dual use; V2 source disables both but still reads ADC `-1` |
| 3 | CC1101 GDO2; generic UART TX | CC1101/optional serial | GPIO/UART | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | mutually exclusive roles |
| 4 | CSN #1 | nRF24 #1 | shared radio SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | must remain high when another slave owns bus |
| 5 | CC1101 CS; optional GPS RX; optional PN532 CS | CC1101/GPS/PN532 | SPI/UART | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | mutually exclusive optional configurations |
| 6 | CC1101 GDO0; optional GPS TX; generic UART RX | CC1101/GPS/serial | GPIO/UART | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | mutually exclusive roles |
| 7 | TFT backlight PWM | display | PWM/GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | owned by display/power UI |
| 8 | SDA | PCF8574 and possible IP5306 | I2C | `CONFIRMED_SCHEMATIC`; source calls `Wire.begin()` without pins | shared open-drain bus; source mapping is implicit |
| 9 | SCL | PCF8574 and possible IP5306 | I2C | `CONFIRMED_SCHEMATIC`; source calls `Wire.begin()` without pins | shared open-drain bus; source mapping is implicit |
| 10 | SD CS | microSD | shared radio/storage SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | exclusive slave selection |
| 11 | MOSI | SD, CC1101, nRF24 | shared radio/storage SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | common signal |
| 12 | SCLK | SD, CC1101, nRF24 | shared radio/storage SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | per-device speed/state |
| 13 | MISO; CC1101 GDO1 on module drawing | SD, CC1101, nRF24 | shared radio/storage SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | clients must release line; GDO1 is not used by firmware |
| 14 | CE #3; IR TX | nRF24 #3/IR | GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | mutually exclusive at runtime |
| 15 | CE #1 | nRF24 #1 | GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | radio owner |
| 16 | TFT DC | display | display SPI control | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | display owner |
| 17 | TFT CS | display | display/touch SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | deselect during touch transaction |
| 18 | touch CS | XPT2046 | display/touch SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | deselect during TFT transaction |
| 19 | native USB D- | ESP32-S3 | USB | `CONFIRMED_SCHEMATIC` | dedicated USB role |
| 20 | native USB D+ | ESP32-S3 | USB | `CONFIRMED_SCHEMATIC` | dedicated USB role |
| 21 | CSN #3; IR RX | nRF24 #3/IR | SPI CS/GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | mutually exclusive at runtime |
| 35 | TFT/touch MOSI | ILI9341/XPT2046 | display/touch SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | shared physical signal |
| 36 | TFT/touch SCLK | ILI9341/XPT2046 | display/touch SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | shared physical signal |
| 37 | TFT/touch MISO | ILI9341/XPT2046 | display/touch SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | shared physical signal |
| 38 | SD card detect assumption | microSD | GPIO input | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | contact/polarity requires runtime verification |
| 47 | CE #2 | nRF24 #2 | GPIO | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | radio owner |
| 48 | CSN #2 | nRF24 #2 | shared radio SPI | `CONFIRMED_SCHEMATIC`, `CONFIRMED_SOURCE` | must remain high when another slave owns bus |

## PCF8574 expanded inputs

V2 firmware assigns expander pins P7/P5/P3/P4/P6 to Up/Down/Left/Right/Select.
The schematic shows five tactile switches and PCF pins P0 through P7 with board-specific
button net labels, but a physical button-to-firmware mapping has not been recorded.
`HW-007` will establish the authoritative mapping.

## Bus topology

### Radio/storage SPI

| Client | SCLK/MOSI/MISO | CS/CE/IRQ | Nominal source configuration | Power | Current initialization owner |
| --- | --- | --- | --- | --- | --- |
| microSD | 12/11/13 | CS 10; detect 38 | mode 0, mount requested at 4 MHz | 3.3 V | `utils.cpp`, plus direct ESB path |
| CC1101 | 12/11/13 | CS 5; GDO0 6; GDO2 3 | library-specific; utility restores bus to 4 MHz | 3.3 V | each sub-GHz feature |
| nRF24 #1 | 12/11/13 | CSN 4; CE 15; IRQ unused | RF24 object requests 16 MHz; raw modes set 10 MHz/mode 0 | 3.3 V, local 10 µF shown | Bluetooth/nRF feature |
| nRF24 #2 | 12/11/13 | CSN 48; CE 47; IRQ unused | RF24 object requests 16 MHz | 3.3 V, local 10 µF shown | multi-radio modes |
| nRF24 #3 | 12/11/13 | CSN 21; CE 14; IRQ unused | raw scanner/ESB/MJ modes set 10 MHz/mode 0 | 3.3 V, local 10 µF shown | selected nRF feature |
| optional PN532 | SCLK 12, source MOSI 13/MISO 11 | CS 5 | software SPI; data roles opposite hardware bus | external/unknown | `rfid.cpp` |

Project source has no shared `SPI.beginTransaction()`/`SPI.endTransaction()` invariant.
Utilities attempt recovery by raising known CS pins, ending SD/SPI, resetting pins, and
remounting. Several features directly initialize or remap the bus.

### Display/touch SPI

| Client | SCLK/MOSI/MISO | CS/control | Power | Current behavior |
| --- | --- | --- | --- | --- |
| ILI9341 TFT | 36/35/37 | CS 17, DC 16, reset discrepancy, BL 7 | logic 3.3 V; backlight circuit uses 5 V path | TFT_eSPI configured for HSPI |
| XPT2046 touch | 36/35/37 | CS 18, IRQ absent in source | 3.3 V | separate `SPIClass(HSPI)` object is begun repeatedly before samples |

The physical signals and likely SPI host are shared, even though source sets
`TOUCH_SHARES_TFT_SPI` to zero and manages touch through a separate object. No explicit
cross-client lock is present.

### I2C

| Client | Address | SDA/SCL | Power | Evidence/status |
| --- | --- | --- | --- | --- |
| PCF8574T | source scans 0x20-0x27 | 8/9 | 3.3 V | physical and schematic confirmed; runtime address unknown |
| IP5306 | unknown/variant-dependent | schematic connects LED2/SDA and LED1/SCL to 8/9 nets | battery/5 V power domain, interface pulled to 3.3 V | marking/schematic confirmed; telemetry support unknown |

`Wire.begin()` uses target defaults rather than explicit pins, then sets a 50 ms timeout.
Only the PCF range is probed. No general I2C scan or IP5306 access exists.

The canonical SD CS is GPIO10, but `shared.h` also retains a legacy `SD_CS_PIN` default
of GPIO5. Utility code tries to reject that value when it collides with CC1101, while
some feature code still contains fallback logic. This is duplicated configuration, not
evidence that the reference SD socket uses GPIO5.

### USB and UART

- One Type-C connector routes native ESP32-S3 USB D-/D+ on GPIO19/20.
- A second Type-C connector routes through the CP2102/CP2102N-family bridge to UART0.
- Optional GPS uses UART2 on GPIO5/6, conflicting with CC1101 control signals.
- Source uses `Serial` for console but does not expose which physical connector is active
  under each Arduino USB build option.

## Ownership requirements derived from the current hardware

1. All non-owner chip-selects must be inactive before a shared transaction.
2. CE must be low for inactive nRF radios unless a feature deliberately owns them.
3. SPI mode, bit order, and frequency must match the selected client.
4. An SD mount or open file must not be invalidated by another task or feature.
5. Display and touch transactions must not overlap or independently reset their host.
6. GPIO14/21 and GPIO5/6 optional functions require an explicit configured owner.
7. Resource restoration must be verified, not assumed from a call sequence.

These invariants justify evaluating a small SPI owner later, but this audit does not
implement one.

## Audited API surface

The source audit covered all project occurrences of `SPI.begin/end/setFrequency`,
transaction calls, `SD.begin/end`, RF24/CC1101 initialization, `Wire`, shared-pin
`pinMode`, and chip-select `digitalWrite`. Direct access is concentrated in
`utils.cpp`, `bluetooth.cpp`, `subghz.cpp`, `Touchscreen.cpp`, and `rfid.cpp`, with SD
clients in settings, Wi-Fi, GPS, IR, Ducky, and sub-GHz modules.
