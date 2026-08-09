# Power and battery knowledge

## Current evidence

| Finding | Evidence | Confidence |
| --- | --- | --- |
| Package is marked `IP5306` | `CONFIRMED_PHYSICAL` | High for marking only |
| Main BOM calls U4 IP5306 | `CONFIRMED_SCHEMATIC` | High for design intent |
| Schematic uses IP5306 for charging/boost/power key path | `CONFIRMED_SCHEMATIC` | High for design intent |
| IP5306 LED1/LED2 pins connect to shared SCL/SDA nets | `CONFIRMED_SCHEMATIC` | High |
| Exact variant supports usable I2C telemetry | `UNKNOWN` | Low |
| Actual assembled PCB routes/responds on I2C | not yet `CONFIRMED_RUNTIME` | Unknown |

The LED pin names and shared net labels are not sufficient to claim I2C telemetry.
They may be multifunction pins only on particular variants or configurations.

## Current firmware lifecycle

Boot starts a core-0 status task. Every 400 ms it calls `readBatteryVoltage()`, maps
3.00-4.20 V linearly to 0-100%, and marks the status bar dirty. The non-task fallback
does the same every two seconds.

For V2, `BATTERY_ADC_PIN` is `-1`. `readBatteryVoltage()` nevertheless calls
`analogSetPinAttenuation(-1, ...)` and repeatedly calls `analogReadMilliVolts(-1)`.
The displayed percentage therefore has no justified measurement source.

The schematic shows a resistor divider associated with VBAT and GPIO2, while a buzzer
driver is also controlled from GPIO2. Source disables both V2 buzzer and battery ADC.
This is a design/source conflict requiring trace and runtime tests; it is not permission
to enable GPIO2 blindly.

There is no source-level handling for:

- IP5306 registers or presence;
- charge state;
- power-path mode;
- power-key/shutdown behavior;
- low-battery shutdown;
- calibrated ADC conversion;
- telemetry validity/unsupported state.

## Required separation in future diagnostics

| Datum | Current state | Valid future presentation |
| --- | --- | --- |
| charging behavior | physically plausible, unmeasured | `NOT TESTED` |
| power path | schematic evidence only | `NOT TESTED` |
| power button/shutdown | user-observable, unlogged | `NOT TESTED` |
| battery voltage | no valid V2 source established | `UNSUPPORTED` or `UNKNOWN` |
| charging state | no implementation | `UNKNOWN` |
| battery percentage | unjustified | do not display |
| IP5306 I2C | shared nets in schematic only | `UNKNOWN` pending safe probe |

## Characterization sequence

1. Record I2C scan with and without battery/USB power; do not write registers.
2. Identify PCF8574 first so another response can be isolated.
3. Trace/measure GPIO2 and battery-divider behavior before configuring ADC or buzzer.
4. Record power-button behavior under USB-only, battery-only, and combined power.
5. With a USB power meter or DMM, measure idle and major receive/scan modes.
6. With suitable current capture, measure peaks during owned-device RF tests without
   changing existing output-power behavior.

See `HW-006` and `PWR-001` through `PWR-003` in `docs/INVESTIGATIONS.md`.
