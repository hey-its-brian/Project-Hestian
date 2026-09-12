# Hestian architecture

Three ESPHome devices and Home Assistant as the hub.

```
                         +-------------------------+
                         |     Home Assistant      |
                         |  climate.hestian_...    |
                         |  config entities        |
                         |  dashboards, automations|
                         +-----+-----------+-------+
                               |           |
              native API       |           |  native API
        (state up, actions down)           |
                               |           |
   +---------------------------+--+     +--+----------------------------+
   |  hestian-dial (M5Stack Dial)  |     |  hestian-remote-1 / -2 (CYD)  |
   |  THE THERMOSTAT               |     |  THIN CLIENTS                 |
   |  ESPHome `thermostat` climate |     |  read climate state from HA   |
   |  heat_cool, dual setpoints    |     |  write setpoints via actions  |
   |  BME680 (temp/RH/pressure)    |     |  SCD40 + BME680 local sensors |
   |  PCF8574 -> relay board       |     |  CO2, IAQ, pressure on screen |
   +-------------+-----------------+     +-------------------------------+
                 |
        24 VAC   |  Y / O/B / W2 / G
                 v
        +------------------------------+
        |  Heat pump air handler       |
        |  + auxiliary electric heat   |
        +------------------------------+
```

## Roles

**hestian-dial** owns the HVAC. The system is a heat pump with auxiliary
electric heat: the compressor (Y) runs for both heating and cooling, the
reversing valve (O/B) picks the direction, the strips (W2/E) come in as a
second heating stage or alone as emergency heat, and G runs the blower.
The thermostat logic runs on the Dial itself,
so heating and cooling keep working if WiFi or Home Assistant is down. It
exposes one `climate` entity plus a set of config entities, and imports the
two remote room temperatures so the control sensor can be switched between
the hallway, either remote, or an average.

**hestian-remote-1 / -2** never touch the HVAC. They subscribe to the climate
entity's attributes over the native API and call `climate.set_temperature` /
`climate.set_hvac_mode` when a button is pressed. If HA is down, the remote
still shows its own room's readings but the thermostat panel goes stale.

**Home Assistant** is the settings surface and the glue. All Dial tuning
(deadbands, cycle timers, sensor offset, sensor source, screen timeout,
display units) lives under the device's Configuration section. No web app is
needed; ESPHome's built-in web server was deliberately left off the Dial
because the S3 has no PSRAM and LVGL plus WiFi already use most of the RAM.

## Why these choices

| Decision | Choice | Reason |
|---|---|---|
| Brain | M5Stack Dial replaces the XIAO C6 from the original plan | Final wall device from day one; one firmware to maintain instead of a migration |
| Dial sensor | BME680, gas heater disabled | Tighter temperature spec than the SCD40, no 200 mA CO2 lamp pulses warming the board, and CO2 matters in occupied rooms (the remotes), not the hallway |
| Relay drive | PCF8574 I2C expander into the 4-channel opto-isolated relay board | Native ESPHome support, three outputs from one Grove port, reuses the already-specced AC-rated relay board. The M5Stack 4-Relay unit needs an unmaintained external component |
| Thermostat mode | `heat_cool` with `target_temperature_low` (heat on) and `target_temperature_high` (AC on) | Exactly the low/high band requested; HA's climate card shows both handles |
| Heat pump staging | ESPHome `supplemental_heating_action` drives W2 | Strips engage when the room is "Aux Heat Delta" below the setpoint or after "Max Compressor Heat Run"; both HA-adjustable |
| Emergency heat | Config switch, not a climate mode | HA's climate entity has no emergency-heat mode for ESPHome; a switch keeps the climate card clean and the Dial shows EM HEAT |
| Reversing valve | Config select O / B | Brand-dependent polarity; changeable without reflashing |
| Tuning | ESPHome `number` entities calling the thermostat's runtime setters | Editable from HA, persisted on the device, re-applied at boot |
| Remote sensors | SCD40 in `low_power_periodic` for CO2 only; BME680 for temp/RH/pressure/gas | Cuts SCD40 self-heating by about 5x; BME680 pressure feeds SCD40 compensation |
| Remote display driver | `mipi_spi` with the `ESP32-2432S028-7789` preset | The 2-USB CYD ships with an ST7789V (or occasionally an ILI9342); `ili9xxx` is deprecated |

## Data flow for a setpoint change from a remote

1. Button tap on the CYD adjusts a pending value locally and redraws.
2. After 1 s of no taps, the CYD calls `climate.set_temperature` on HA with
   `target_temp_low` / `target_temp_high` (or `temperature` in heat-only or
   cool-only mode).
3. HA forwards the call to the Dial over the native API.
4. The Dial's thermostat updates, publishes new state, and the Dial screen
   redraws.
5. HA pushes the new attributes back to both remotes, which redraw.

## Units

The Dial's thermostat works in Celsius internally (ESPHome convention). HA
converts to the system's unit. The Dial screen converts for display based on
the "Display Units" select. The remotes show whatever HA hands them, so set
`units_label` in each remote file to match HA.

## Firmware layout

```
esphome/
  hestian-dial.yaml           the thermostat
  hestian-remote-1.yaml       per-room overrides only
  hestian-remote-2.yaml
  packages/
    common.yaml               wifi / api / ota / diagnostics for every node
    remote-cyd.yaml           the whole remote config, parameterised
  secrets.yaml.example        copy to secrets.yaml
```
