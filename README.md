# Project-Hestian

A fully self-made IoT home thermostat replacement based on the ESP32 platform and ESPHome, controlled through Home Assistant.

Named for Hestia, Greek goddess of the hearth.

## Overview

Hestian replaces a basic 24 VAC digital thermostat with a custom ESP32-based controller. A Seeed Studio XIAO ESP32-C6 reads room temperature from a BME280 sensor and switches the heat, cool, and fan calls through an opto-isolated relay board. ESPHome's `thermostat` climate platform handles the hysteresis and cycle-protection logic and exposes a standard `climate` entity to Home Assistant.

## Hardware

- Seeed Studio XIAO ESP32-C6 (controller)
- BME280 temperature, humidity, and pressure sensor (I2C)
- 4-channel opto-isolated relay module (switches the 24 VAC calls)
- 5V USB-C power supply for the logic side
- Printed enclosure (Bambu P1S)

The relays only switch the 24 VAC that comes from the air handler's own transformer. The logic side is powered separately over USB, which avoids the C-wire and power-stealing complications.

## Phases

1. Headless thermostat controlled from the Home Assistant dashboard
2. (Phase 1.5) CYD wall panel as a physical view and control surface
3. (Phase 2) Onboard status display on the thermostat itself
4. (Phase 3) M5Stack Dial knob controller as the final wall device

## Status

Early build. Planning is complete and hardware is being gathered. See the project notes for the full bill of materials, wiring guide, and per-phase trackers.

## License

See [LICENSE](LICENSE).
