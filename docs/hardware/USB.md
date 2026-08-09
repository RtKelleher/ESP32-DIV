# USB and serial knowledge

## Physical and schematic topology

The reference board physically contains a Silicon Labs CP2102N-family USB-UART device.
The supplied schematic/BOM names the design component generically as CP2102. Exact
package suffix and identity beyond the observed family marking are not recorded.

The V2 schematic shows two Type-C paths:

1. native ESP32-S3 USB D-/D+ on GPIO19/20;
2. a CP2102/CP2102N-family bridge connected to ESP32 UART0, with DTR/RTS transistor
   control for reset/boot behavior.

Which physical connector is labeled USB1/USB2 on the assembled enclosure/PCB has not
been recorded as `CONFIRMED_RUNTIME`.

## Current firmware/build behavior

- Firmware calls `Serial.begin(115200)` for the console.
- GPS uses a separate UART2 instance on GPIO5/6 when enabled.
- The patched Arduino platform forwards USB mode and CDC-on-boot board options.
- The reconstructed baseline used the ESP32-S3 hardware-CDC USB mode selection with
  the repository's current board options.
- Firmware does not report whether `Serial` currently maps to UART0/CP2102 or native
  USB CDC, nor whether both recovery paths remain available.

## Why both paths matter

The CP2102N-family path is part of the reference architecture and provides a valuable
independent UART/reset/boot route. Native USB offers separate ESP32-S3 capabilities.
Neither should be removed or conflated during build migration.

## HW-008 test

For each connector, record:

- USB VID/PID and host device name;
- whether a serial port enumerates before and after firmware boot;
- whether boot logs appear and at what baud/settings;
- reset behavior when the serial port is opened;
- upload/recovery behavior with BOOT/RESET;
- behavior with native USB CDC enabled and disabled;
- which path remains usable after a bad application image.

No connector should be assigned a definitive USB1/USB2 role until this is logged.
