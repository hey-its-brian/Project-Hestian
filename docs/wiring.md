# Hestian wiring

> **Before touching the HVAC side:** 24 VAC will not shock you, but shorting
> R to C blows the air handler's low-voltage fuse. Kill power to the air
> handler at its switch or breaker before landing any wire on the relay
> outputs. Photograph and label the old thermostat's terminals first and keep
> the old thermostat as a known-good fallback.

## 1. Dial thermostat

Two firmware variants share the same sensor, display, knob and menu:

| File | System | Relay drive | Status |
|---|---|---|---|
| `esphome/hestian-dial-heatpump.yaml` | Heat pump + aux strips (this apartment) | MCP23017 on PORT.A, 4 relays | Validated, not yet hardware tested |
| `esphome/hestian-dial-conventional.yaml` | Furnace / air handler W + AC on Y | PORT.B GPIO1/2 direct, 2 relays | Bench tested 2026-10-04 |

`esphome/hestian-dial-lvgl-draft.yaml` is the superseded 2026-09-12 draft and
is not for flashing (its PORT.A pins and parts are wrong).

### Pins confirmed on the bench (2026-10-04)

M5Stack's Grove colours did not match the expected GPIOs on either port.
Always confirm in software.

| Port | Wire | GPIO | Role |
|---|---|---|---|
| PORT.A | red / black | | 5 V / GND |
| PORT.A | | **GPIO15** | **SDA** |
| PORT.A | | **GPIO13** | **SCL** |
| PORT.B | red / black | | 5 V / GND |
| PORT.B | white | GPIO1 | relay IN1 (conventional build) |
| PORT.B | yellow | GPIO2 | relay IN2 (conventional build) |

Both Grove ports supply **5 V only**. The Dial's I2C pins are not 5 V
tolerant, so everything on PORT.A runs from a buck converter set to 3.3 V
(measure it before connecting anything; adjustable modules often ship set
high).

### Relay board

DZS Elec 4-channel opto-isolated board, 5 V coils, powered from a Grove 5 V
rail. Set each used channel's trigger jumper to **H** (high-level trigger):
3.3 V turns the relay on, 0 V turns it off, and a booting Dial holds it off.
No `inverted: true` in the firmware.

### Heat pump build (MCP23017)

Parts: SCD40 breakout with onboard pull-ups, Waveshare MCP23017 I/O
expansion board, 5 V to 3.3 V buck, relay board, Grove-to-Dupont cables.

```
Dial PORT.A                         3.3 V rail / I2C bus
  red 5V  ----- buck IN+   buck OUT+ ---+---- SCD40 VCC
  black GND --- buck IN-   buck OUT- ---+---- SCD40 GND ---- MCP23017 GND
                                        +---------------------- MCP23017 VCC
  GPIO15 (SDA) ---------------------------- SCD40 SDA ------ MCP23017 SDA
  GPIO13 (SCL) ---------------------------- SCD40 SCL ------ MCP23017 SCL

MCP23017          Relay board (all four jumpers on H)
  GPA0 ---------- IN1   Y    compressor (heat and cool)
  GPA1 ---------- IN2   O/B  reversing valve
  GPA2 ---------- IN3   W2/E aux and emergency heat strips
  GPA3 ---------- IN4   G    blower

Dial PORT.B red 5V / black GND ---- relay VCC / GND
```

Grounds must be common (non-isolated buck: IN- and OUT- are the same net).
The MCP23017 address is set in the firmware (0x27 assumed for the Waveshare
board); use whatever the boot log's I2C scan reports. Its pins float as
inputs for a moment at power-up: if any relay chatters at boot, add 10 kΩ
pull-downs from IN1 to IN4 to GND.

The MCP23017 replaces the PCF8574 from the 2026-09-12 plan: the PCF8574 can
only sink current, which would force active-low relays at 5 V logic and put
5 V on the Dial's I2C pins. The MCP23017 has push-pull outputs and runs at
3.3 V.

### Conventional build (direct GPIO)

```
Dial PORT.B                Relay board (jumpers 1 and 2 on H)
  white  GPIO1 ----------- IN1   W  heat
  yellow GPIO2 ----------- IN2   Y  cool (add G here if the blower does not
                                       start on Y by itself)
  red    5V    ----------- VCC
  black  GND   ----------- GND

Dial PORT.A -> buck (3.3 V) -> SCD40, SDA GPIO15, SCL GPIO13 (as above)
```

### 24 VAC side (air handler powered OFF)

> **Before touching the HVAC side:** shorting R to C blows the air handler's
> low-voltage fuse. Kill the air handler at its switch or breaker first.
> Use COM + NO on each relay so every circuit is open at rest.

**Heat pump (this apartment).** Old Emerson stat: W2 (white), E, O/B
(orange), R (red), G (green), C (blue), Y (yellow). Label each wire as it
comes off.

```
Thermostat wires                    Relay board screw terminals
  R  ---+------------------------ COM1
        +------------------------ COM2
        +------------------------ COM3
        +------------------------ COM4
  Y  --------------------------- NO1   compressor (heat and cool)
  O/B ------------------------- NO2   reversing valve
  W2 --------------------------- NO3   aux heat strips
  E  --------------------------- NO3   (with W2, if E is a separate wire)
  G  --------------------------- NO4   blower
  C  ------ not used, cap it (logic side is USB powered)
```

The reversing valve is **O type** (energised to cool): metered 24 VAC
between O/B and C during a cool call on the old stat (2026-09-12). The
heat pump firmware assumes O. If heat ever blows cold, the valve logic is
inverted; nothing breaks, but fix it before leaving it running.

**Conventional system.**

```
  R  ---+------------------------ COM1
        +------------------------ COM2
  W  --------------------------- NO1   heat
  Y  --------------------------- NO2   cool
  G  --------------------------- NO2   with Y, only if the blower needs it
  C  ------ not used, cap it
```

Never land 24 VAC on the Dial's green 6 to 36 V terminal: it is DC only.

### Power

USB-C into the Dial, bench and wall. A 1 A supply is plenty (four relay
coils at about 70 mA each plus the Dial).

### Sensor placement

The SCD40 reads high when it sits near the Dial and buck (bench: 74.8 °F
drifting to 79.6 °F against a 75.2 °F reference). Mount it outside and
below the enclosure with an air gap, then set `temperature_offset` against a
reference thermometer once it is in its final position.

## 2. Remote units (CYD)

### Parts (per unit)

- ESP32-2432S028R, 2-USB variant
- SCD40 breakout
- BME680 breakout
- 4-pin 1.25 mm JST/PicoBlade pigtail for CN1
- USB-A to micro-USB or USB-A to USB-C cable (the USB-C port lacks CC
  resistors, so a C-to-C cable will not power the board)

### CN1 connector

| CN1 pin | Signal | Use |
|---|---|---|
| 1 | GND | both sensors |
| 2 | GPIO22 | SCL |
| 3 | GPIO27 | SDA |
| 4 | 3.3 V | both sensors |

```
CN1                 SCD40            BME680
  GPIO22 (SCL) ---- SCL ------------ SCL
  GPIO27 (SDA) ---- SDA ------------ SDA
  3V3 ------------- VDD ------------ VIN
  GND ------------- GND ------------ GND
```

Addresses: SCD40 is fixed at 0x62; BME680 is 0x76 or 0x77 (set
`bme680_address` in the device file after checking the boot log's I2C scan).

Do not use P3 for I2C: its GPIO21 is the backlight and GPIO35 is input only.

### Placement

- Sensors outside the CYD's case or in a vented pocket away from the display.
- Not in direct sun, not in the HVAC vent stream.

## 3. Bring-up order

1. **Dial on the bench.** Flash over USB-C with a data cable. Add
   `hestian_api_key` (`openssl rand -base64 32`) and `hestian_ota_password`
   to `secrets.yaml`. Confirm screen, knob and button, and adopt it in HA.
2. **Buck and SCD40.** Set the buck to 3.3 V on a meter first. The boot
   log's I2C scan should show 0x62. "SCL is held low" or "Found no devices"
   means no pull-ups or swapped SDA/SCL.
3. **MCP23017 and relay board (logic side only).** The scan should show the
   expander's address; set it in the firmware. Relays are internal to the
   thermostat, so test them through modes, and expect the 5 minute startup
   delay after every flash:
   - Heat, setpoint a little above room temp: Y and G click on.
   - Heat, setpoint 5 °F or more above room temp: W2 joins (aux).
   - Emergency Heat on in HA during a heat call: Y drops, W2 and G on.
   - Cool, setpoint below room temp: O/B, Y and G on.
   - Off: everything releases, including O/B.
   Meter COM to NO on each energised relay.
4. **Wall install.** Air handler off. Move the wires per the 24 VAC diagram.
   Power up. Test cool (outdoor unit runs, cold air), then heat (warm air;
   if cold, the valve logic is inverted), then Emergency Heat (strips only,
   outdoor unit off).
5. **Calibrate and tune.** Temperature offset against a reference, then
   deadbands and cycle timers if needed.
6. **Remotes.** Flash, confirm sensors in HA, enable "Allow the device to
   perform Home Assistant actions" on each remote in the ESPHome integration,
   then test a setpoint change from each screen.
