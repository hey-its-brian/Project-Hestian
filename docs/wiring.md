# Hestian wiring

> **Before touching the HVAC side:** 24 VAC will not shock you, but shorting
> R to C blows the air handler's low-voltage fuse. Kill power to the air
> handler at its switch or breaker before landing any wire on the relay
> outputs. Photograph and label the old thermostat's terminals first and keep
> the old thermostat as a known-good fallback.

## 1. Dial thermostat

### Parts

- M5Stack Dial v1.1
- BME680 breakout (I2C)
- PCF8574 I2C expander breakout (address 0x20 with A0..A2 low)
- 4-channel opto-isolated 5 V relay module (active low inputs); all four
  channels are used
- Grove cable with one end cut to bare leads, or a Grove-to-Dupont cable
- 5 V USB-C supply for the Dial (bench and wall)

### Grove PORT.A pinout (looking at the Dial's port)

| Grove pin | Signal | GPIO |
|---|---|---|
| 1 (yellow) | SCL | GPIO15 |
| 2 (white) | SDA | GPIO13 |
| 3 (red) | 5 V | |
| 4 (black) | GND | |

PORT.A carries 5 V, not 3.3 V. Both the BME680 breakout and the PCF8574
breakout tolerate 5 V supply on their VIN pins (they have onboard regulators
or are 5 V parts). The I2C lines idle at 3.3 V from the Dial's pull-ups; the
PCF8574 is fine with that, and the BME680 breakout's level shifter handles it.

### I2C bus (all in parallel on PORT.A)

```
Dial PORT.A          BME680           PCF8574
  SCL  ------------- SCL ------------ SCL
  SDA  ------------- SDA ------------ SDA
  5V   ------------- VIN ------------ VCC
  GND  ------------- GND ------------ GND
                                       A0, A1, A2 -> GND   (address 0x20)
```

Keep the run from the Dial to the BME680 short and mount the sensor below
the Dial in a vented pocket. The Dial's display and radio warm its body by a
degree or two; the sensor must not sit in that plume.

### PCF8574 to relay board

```
PCF8574        Relay board
  P0  -------- IN1   (Y, compressor)
  P1  -------- IN2   (O/B, reversing valve)
  P2  -------- IN3   (W2/E, aux and emergency heat strips)
  P3  -------- IN4   (G, blower)
  VCC -------- VCC   (5 V, shared from PORT.A)
  GND -------- GND
```

The PCF8574's outputs are weak high, strong low. That pairs correctly with
an active-low relay board: the expander sinks current to pull an input low
and energise a relay. At power-up every PCF8574 pin is high, so all relays
start off. ESPHome sets `inverted: true` on each switch so "on" in HA means
"relay closed".

### 24 VAC side (air handler powered OFF)

The system is a **heat pump with auxiliary electric heat**. The old Emerson
thermostat had these wired: W2 (white), E, O/B (orange), R (red), G
(green), C (blue), Y (yellow). Confirm each colour against its terminal as
you remove them and label them.

Use COM + NO on each relay so every circuit is open at rest.

```
Thermostat wires                    Relay board screw terminals
  R  ---+------------------------ COM1
        +------------------------ COM2
        +------------------------ COM3
        +------------------------ COM4
  Y  --------------------------- NO1   compressor (heat and cool)
  O/B ------------------------- NO2   reversing valve
  W2 --------------------------- NO3   aux heat strips
  E  --------------------------- NO3   (same terminal as W2; see below)
  G  --------------------------- NO4   blower
  C  ------ not used (logic side is USB powered)
```

**W2 and E.** On this air handler, W2 (second-stage / auxiliary heat) and E
(emergency heat) both end up energising the strip heaters. If the old stat
had a separate E wire, land it on NO3 with W2; if E was only a jumper on the
old stat, there is nothing to land. The firmware's "Emergency Heat" switch
runs the strips with the compressor off, which is what the EMER position
did.

**O or B.** This system is **O type** (valve energised to cool): metered
24 VAC between O/B and C during a cool call on the old thermostat. The
firmware's "Reversing Valve" select defaults to O, so leave it. If the
select is ever wrong, heat blows cold; nothing breaks, but fix it before
leaving it running.

### Power

Bench: USB-C into the Dial. Wall: the same. A 1 A wall wart is plenty; the
four relay coils draw about 70 mA each. The Dial's 6 to 36 V DC terminal is
an alternative if a DC supply is handier behind the wall plate. Whether the
Grove 5 V rail is powered from the DC terminal input was not verifiable from
M5Stack's docs, so confirm with a meter before relying on it.

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

1. **Dial on the bench, nothing on PORT.A.** Flash over USB-C with a data
   cable (a charge-only cable shows no USB device at all). Do not hold the
   boot button under the back sticker; if you did, press reset afterwards or
   the chip stays in download mode and the app never starts. Expect
   "PCF8574 not available" and "bme680 marked as failed" in the log until
   step 2. Confirm the screen, encoder, button and touch work and the device
   appears in HA. Serial logs come out of the USB-C port; use
   `esphome logs hestian-dial.yaml` after pressing reset if it shows nothing.
2. **Add the BME680 and PCF8574.** Check the boot log's I2C scan shows both
   addresses. Confirm "Hallway Temperature" reads sanely in HA.
3. **Add the relay board (logic side only).** From the HA device page toggle
   Relay Compressor / Reversing Valve / Aux Heat / Fan. Each relay should
   click and its LED light. Verify
   COM to NO continuity on the energised relay with a meter.
4. **Exercise the thermostat on the bench.** Set the heat setpoint above room
   temperature in heat_cool mode and watch the heat relay close after the
   idle timer. Lower it and watch it open after the min run time.
5. **Wall install.** Air handler off. Move the thermostat wires to the relay
   board per the 24 VAC diagram. Set "Reversing Valve" in HA for the brand.
   Power up. Call for fan first (lowest risk), then cool (outdoor unit runs,
   cold air at the registers), then heat (warm air; if it blows cold, flip
   the Reversing Valve select), then Emergency Heat (strips only, outdoor
   unit stays off).
6. **Tune** the deadbands and cycle timers from the HA device page.
7. **Remotes.** Flash, confirm sensors in HA, enable "Allow the device to
   perform Home Assistant actions" on each remote in the ESPHome integration,
   then test a setpoint change from each screen.
