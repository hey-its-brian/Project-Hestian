# Project Hestian

A self-built smart thermostat for a 24 VAC heat pump with auxiliary electric
heat, running ESPHome and controlled through Home Assistant. Named for Hestia, Greek goddess
of the hearth.

## What it is

- **Wall thermostat:** an M5Stack Dial (ESP32-S3, round touchscreen, rotary
  ring). It runs the thermostat logic itself (and keeps running if HA or
  WiFi is down), reads room temperature from an SCD40, and switches the
  compressor, reversing valve, aux heat strips and blower through an
  opto-isolated relay board via an MCP23017 I2C expander. Twist the ring to
  move the active setpoint in Heat or Cool mode; press for a menu (mode,
  setpoints, brightness). A conventional W/Y variant is included too.
- **Two remote units:** Cheap Yellow Displays (ESP32-2432S028R, 2-USB) with an
  SCD40 and a BME680 each. Each shows the thermostat's temperature, mode and
  both setpoints with +/- buttons, plus the local room's temperature,
  humidity, CO2, air-quality index and pressure.
- **Home Assistant** is the hub and the settings surface. The thermostat is a
  `climate` entity in heat_cool mode with a low setpoint (heat turns on) and a
  high setpoint (AC turns on). Deadbands, cycle timers, sensor offset, which
  room's sensor drives the thermostat, and display options are all config
  entities on the device page.

## Repository layout

```
esphome/                ESPHome firmware
  hestian-dial-heatpump.yaml      the thermostat, heat pump + aux (this apartment)
  hestian-dial-conventional.yaml  the thermostat, W heat + Y cool (bench tested)
  hestian-dial-lvgl-draft.yaml    superseded 2026-09-12 draft, reference only
  hestian-remote-1.yaml   remote unit, living room
  hestian-remote-2.yaml   remote unit, bedroom
  packages/common.yaml    shared wifi / api / ota / diagnostics
  packages/remote-cyd.yaml  full remote config, parameterised per room
  secrets.yaml.example    copy to secrets.yaml
docs/
  architecture.md         roles, data flow, why each choice was made
  wiring.md               Dial, relay, 24 VAC side, CYD sensors, bring-up order
  home-assistant.md       adopting devices, permissions, where settings live
homeassistant/
  packages/hestian.yaml   optional away preset + CO2 warning automations
  dashboards/hestian.yaml starter Lovelace view
```

## Quick start

```bash
cd esphome
cp secrets.yaml.example secrets.yaml   # fill in wifi, api key, ota password
esphome run hestian-dial-heatpump.yaml # first flash over USB-C
esphome run hestian-remote-1.yaml
esphome run hestian-remote-2.yaml
```

Then follow the bring-up order in `docs/wiring.md` before touching the HVAC
wiring, and `docs/home-assistant.md` to adopt the devices and allow the
remotes to call actions.

## Hardware

| Unit | Board | Sensors | Other |
|---|---|---|---|
| Thermostat | M5Stack Dial v1.1 | SCD40 on Grove PORT.A via a 3.3 V buck | MCP23017 expander, 4-ch 5 V relay board (H trigger), USB-C supply |
| Remote x2 | ESP32-2432S028R (2-USB CYD) | SCD40 + BME680 on CN1 | on-board 2.8" touch display |

The relays only switch the 24 VAC that comes from the air handler's own
transformer. All logic is USB powered, so there is no C-wire dependency.

## Status

2026-10-04: conventional Dial firmware bench tested end to end (sensor,
heat/cool/auto, cycle protection, startup delay, interlock, knob and menu
UI). The heat pump firmware is the same build with a four-relay MCP23017
layer; it is validated and waiting on the expander for a bench test. Wall
install follows. The CYD remote configs still assume the 2026-09-12 plan. Detailed phase tracking lives in the Obsidian vault
(Hestian project folder).

## License

See [LICENSE](LICENSE).
