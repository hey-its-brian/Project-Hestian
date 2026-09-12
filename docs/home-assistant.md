# Home Assistant setup

## Adopting the devices

Each node announces itself over mDNS once it joins WiFi. In HA go to
Settings, Devices and Services, and accept the discovered ESPHome devices.
Paste the API encryption key from `esphome/secrets.yaml` when asked.

## Allow the remotes to call actions

The remotes change the thermostat by calling `climate.set_temperature` and
`climate.set_hvac_mode`. HA blocks this by default. For each remote:

1. Settings, Devices and Services, ESPHome.
2. Open the remote's entry, click Configure.
3. Tick "Allow the device to perform Home Assistant actions".

The Dial does not need this; it only reads the remote temperature sensors.

## Entity names

The thermostat entity is `climate.hestian_thermostat` (device "Hestian",
entity "Thermostat"). If HA names it differently, update `thermostat_entity`
in both remote files.

The Dial imports remote temperatures from
`sensor.hestian_remote_1_temperature` and `sensor.hestian_remote_2_temperature`.
Those come from the remotes' BME680 "Temperature" sensors. If the room names
produce different entity ids, update `remote1_temp_entity` and
`remote2_temp_entity` in `hestian-dial.yaml`.

## Where the settings live

Open the Hestian device page. Under **Configuration**:

| Entity | What it does |
|---|---|
| Temperature Offset | Added to the BME680 reading. Calibrate against a trusted thermometer once the Dial is on the wall. |
| Sensor Source | Dial, Remote 1, Remote 2, or Average. Which reading the thermostat regulates on. Falls back to the Dial if a remote is unavailable. |
| Setpoint Minimum Gap | Smallest allowed gap between the heat and cool setpoints. |
| Heat / Cool Deadband | How far below (heat) or above (cool) the setpoint the room must drift before a call starts. |
| Heat / Cool Overrun | How far past the setpoint the call runs before stopping. |
| Min Heat / Cool Run Time | Once started, a call runs at least this long. Both protect the compressor, which runs for heating too. |
| Min Heat / Cool Off Time | Once stopped, a call waits at least this long before restarting. |
| Min Idle Time | Minimum pause between any two calls. |
| Reversing Valve | O (energised to cool, most brands) or B (energised to heat, Carrier and Bryant). |
| Aux Heat Delta | How far below the heat setpoint the room must be before the electric strips join the compressor. |
| Max Compressor Heat Run Before Aux | If the compressor alone has been heating this long without satisfying, bring in the strips. |
| Emergency Heat | Strips only, compressor off. Same as the old thermostat's EMER position. Use when the outdoor unit is iced, failed, or being serviced. |
| Display Units | What the Dial screen shows. HA's own display follows the HA unit system. |
| Screen Timeout / Brightness | Backlight behaviour. |
| Encoder Click Sound | Buzzer tick per detent. |

Under **Diagnostic**: the four relay switches (bench testing only), WiFi
signal, uptime, IP.

Presets Home / Away / Sleep are on the climate entity. Their setpoints are
compiled in (`hestian-dial.yaml`, `preset:` block) because ESPHome does not
expose preset temperatures at runtime; an automation can instead call
`climate.set_temperature` directly if schedule-driven bands are wanted.

## Dashboard

`homeassistant/dashboards/hestian.yaml` is a starter view: the thermostat
card with both handles, the three rooms' temperature, humidity, CO2 and IAQ,
and the config entities in a collapsed section. Paste it into a new
dashboard in YAML mode, or copy individual cards.

## Optional package

`homeassistant/packages/hestian.yaml` adds:

- An input_boolean "Hestian Away" and an automation that switches the
  thermostat preset between Home and Away when it is toggled. Wire the
  boolean to presence however you like.
- A CO2 warning notification when any remote passes 1500 ppm.

Enable packages in `configuration.yaml`:

```yaml
homeassistant:
  packages: !include_dir_named packages
```
