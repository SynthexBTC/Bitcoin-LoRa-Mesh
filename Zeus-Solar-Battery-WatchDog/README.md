# ZEUS : Solar Battery WatchDog

**Battery protection and power-cycle reboot for solar boards**

<img src="images/Zeus Solar Battery WatchDog - 3D Top A.png" height="400">

Battery **voltage** and **temperature** watchdog for a solar-powered board (called the **main board** below), based on a PIC16F13115 microcontroller.

The WatchDog board sits between the solar panel, the Li-ion cell and the main board, which carries the solar charger.

Firmware version: **v10.0.X** (WatchDog board settings)

## What it does

This little module keeps an eye on your battery and temperature, and protects them automatically.

About every 17 seconds, it wakes up, checks the battery voltage and the temperature, acts if needed, then goes back to sleep.

It controls the main board with three switches:

| Output | Switch | Effect |
|---|---|---|
| `CHRG_EN` | Solar panel | **Charging off** = the solar panel is disconnected from the main board |
| `BAT_EN` | Li-ion cell | **Battery off** = the cell no longer powers the main board |
| `RESET_CPU` | Main board power | **Reset** = the main board is completely unpowered for `RESET_PULSE_MS` (5 s) and its capacitors are discharged, then it is powered again |

The reset needs **no wire** to the main board's processor: cutting the main board ground removes both the battery and the solar panel, so the whole main board restarts cleanly, with or without sun.

### Connections

| Connector | Goes to |
|---|---|
| `SOL_IN` | Solar panel |
| `SOL_NODE` | Main board solar input |
| `LI-ION` | Li-ion cell |
| `BAT_NODE` | Main board battery input |

⚠️ On the WatchDog, pin 1 of `SOL_NODE` and `BAT_NODE` is **−** (`GND_NODE`). Check the polarity of the main board connectors and wire **+ to +**, not pin 1 to pin 1, before the first power-up.

### Charging while the battery is off

When the battery is off, the main board charger can still **recharge the cell**. Nothing to do: a deeply discharged cell recovers by itself in the sun, and the battery is switched back on as soon as its voltage exceeds `BAT_OK_MV`.

While the battery is off, the cell can only be charged up to about **3.5 V**. This is why `BAT_OK_MV` is limited to **3500 mV** (default **3400 mV**): a higher value would never be reached, and the battery would never be switched back on.

---

## How it works

### At startup

1. The battery and charging are switched **off**.
2. The module looks for a DS18B20 temperature sensor (see [Temperature sensor](#temperature-sensor)). If there is none, the serial link is started and the version LED, if fitted, blinks the version.
3. After `BAT_CUT_PULSE_MS` (0 by default: no extra wait), the module takes a first measurement.
4. If the temperature and the voltage are OK, the battery is switched **on** and the main board receives a 5 s reset pulse (`RESET_PULSE_MS`). The main board is powered at the end of the pulse. Otherwise the battery stays off until conditions are back to normal. If the temperature cannot be measured, see [Temperature sensor failure](#temperature-sensor-failure).
   A protection switched off (`TEMP_PROT` or `VOLT_PROT`, see below) is not checked here.

### Battery protection (voltage)

| Condition | Action |
|---|---|
| Voltage below `BAT_LOW_MV` (3.2 V) for `BAT_LOW_S` (10 min) | Battery **off** |
| Voltage above `BAT_OK_MV` (3.4 V) for `BAT_OK_S` (immediate) | Battery **on** + reset pulse |

### Charging control

Charging (solar panel connected) is **allowed** only when all of these are true:

- the temperature is between `TEMP_LOW_C` and `TEMP_HIGH_C` (0 °C to 47 °C), or the temperature measurement has failed (see below);
- the voltage has not exceeded `BAT_HIGH_MV` (4.28 V) for `BAT_HIGH_S` (immediate);
- the voltage measurement is working.

**Temperature measurement failure:** charging stays **allowed**, so that the battery can always be recharged. For example, when the battery is very low the sensor may stop working. Charging is then controlled by the voltage rules only: it stops above `BAT_HIGH_MV`. Exception: if a dangerous temperature was measured just before the failure, charging stays off (see [Temperature sensor failure](#temperature-sensor-failure)).

A protection switched off removes its own conditions from this list (see [Disabling a protection](#disabling-a-protection)).

After an overvoltage, charging comes back once the voltage has been back at or below `BAT_HIGH_MV` for `BAT_HIGH_S`.

### Thermal protection

| Condition | Action |
|---|---|
| Temperature at or above `TEMP_DANGER_HIGH_C` (65 °C) | **Total shutdown**: battery and charging off |
| Temperature at or below `TEMP_DANGER_LOW_C` (-20 °C) | **Total shutdown**: battery and charging off |

The total shutdown has **priority over everything else**. It ends when the temperature is back below 62 °C and above -17 °C (`TEMP_DANGER_HYST_C` = 3 °C margin), so the board is not switched on and off repeatedly around a threshold. The battery is then switched back on with a reset pulse, unless the voltage is too low.

### Temperature sensor

The module works with one of two temperature sources. The choice is made **once, when the PIC starts**.

| Mode | Chosen when | Temperature source | Serial link |
|---|---|---|---|
| **DS18B20** | A DS18B20 is detected at startup | DS18B20, measured where it is mounted, e.g. on the cell | **Off**: no `GET` / `SET`, no version LED |
| **UART** | No DS18B20 at startup | PIC internal sensor + `TEMP_OFFSET_C` | On |

- The internal sensor measures the temperature of the **PIC chip**, not of the cell, and is less accurate. For a real thermal protection of the cell, use the DS18B20.
- `TEMP_OFFSET_C` applies to the internal sensor only, **never to the DS18B20**.
- If the DS18B20 cannot be read (for example, sensor unplugged), the internal sensor is used for that check, and the DS18B20 is tried again at the next check.
- In DS18B20 mode the settings are the saved ones (or the defaults). Configure the module in UART mode first, then plug in the DS18B20 and restart the PIC.
- **Changing mode**: a sensor plugged in or unplugged later is ignored until the PIC restarts. To restart the PIC, briefly connect ICSP pin 1 (MCLR) to pin 3 (GND), or disconnect and reconnect the cell. The periodic reboot and the reset pulse do **not** restart the PIC.

**DS18B20 wiring** (on the ICSP port):

| DS18B20 | ICSP port |
|---|---|
| VDD | pin 2 (must be connected) |
| DQ | pin 4, with a **4.7 kΩ** pull-up resistor from DQ to VDD |
| GND | pin 3 |

- Keep the wiring short.
- Put the pull-up resistor on the sensor side of the connector.
- Never connect the DS18B20 and the USB-serial adapter at the same time.
- Unplug the DS18B20 (and its pull-up) before programming.
- The version LED prevents the DS18B20 from being detected: remove its jumper.

### Temperature sensor failure

If the temperature cannot be measured (`TEMP_C : ERR`) for a while, the module continues **as if `TEMP_PROT` were `OFF`**: only the voltage rules apply, and the battery is switched back on with a reset pulse if the voltage allows it.

| Before the failure | Delay before working without temperature |
|---|---|
| Normal temperature, or failure from startup | about 51 s (3 checks) |
| Dangerous temperature measured | about 1 hour, to let the battery cool down |

- Your `TEMP_PROT` setting is **not changed**: as soon as the temperature can be measured again, the temperature protection works again by itself.
- The delays are counted on the regular ~17 s checks only: they are paused while you send commands on the serial link.

### Periodic reboot

Optional: every `REBOOT_D` days, the module power-cycles the main board. It cuts the battery for `BAT_CUT_PULSE_MS` (0 by default), switches it back on, then sends the 5 s reset pulse (main board completely unpowered for 5 s). Disabled by default (`REBOOT_D` = 0). The reboot is postponed while the battery is off for protection.

### Disabling a protection

The voltage and temperature protections can each be switched off, for testing or for a special installation. Both are **ON** by default, and the setting is saved like any other.

| Command | Effect |
|---|---|
| `SET TEMP_PROT OFF` | **All** temperature rules are ignored: total shutdown and charging window (`TEMP_LOW_C` … `TEMP_HIGH_C`) |
| `SET VOLT_PROT OFF` | **All** voltage rules are ignored: `BAT_LOW_MV` cut, `BAT_OK_MV` condition, `BAT_HIGH_MV` charging cut, and charging cut on a faulty voltage measurement |
| `SET TEMP_PROT ON` / `SET VOLT_PROT ON` | The protection is active again |

- Switching a protection **off** acts at once: if nothing else keeps the battery off, it is switched **on** with a reset pulse, and charging is updated.
- The two protections are independent. Example: with `VOLT_PROT OFF` during a thermal shutdown, the battery stays off until the temperature is back to normal.
- Switching a protection back **on** applies its rules again from the next check, with their normal delays (for example `BAT_LOW_S` before a low-voltage cut).
- `GET STATUS` shows the state of both protections.

⚠️ With **both** protections off, the battery and charging stay **on** permanently, with **no supervision at all**.

<img src="images/Zeus Solar Battery WatchDog - SBMS Watchdog Diagram Reset.jpg">

<img src="images/Zeus Solar Battery WatchDog - SBMS Watchdog Diagram VDD.jpg">

<img src="images/Zeus Solar Battery WatchDog - SBMS Watchdog Diagram Temp.jpg">

### Good to know

- **Timing**: the module checks every ~17 s, so all durations are rounded up to a multiple of 17 s and are approximate. They are never shorter than requested. A value of 0 acts immediately.
- **Power consumption**: very low, the chip sleeps almost all the time.
- **While the PIC is not running** (power-up, programming), the battery and the solar panel stay **connected** to the main board, without supervision, until the program starts and switches them off.

---

## Version LED

An optional LED (with a 1 kΩ resistor, between ICSP pin 4 and GND) can be connected through a **jumper**. At startup it blinks the version with short flashes.

- UART mode only: no blink when a DS18B20 is detected.
- Fit the jumper only to check the version, then remove it. Otherwise the LED stays lit and drains the battery, and it prevents the DS18B20 from being detected.
- Do not fit it while a programmer or the USB-serial adapter is connected.

---

## Configuration (serial link)

The serial link is available in **UART mode only**: no DS18B20 connected when the PIC starts (see [Temperature sensor](#temperature-sensor)).

### Connection

Configuration is done through the module's ICSP port, connected to a USB-to-serial adapter (3.3 V logic).

| ICSP port | ↔ | USB-serial adapter |
|---|---|---|
| pin 1 = MCLR | | not connected |
| pin 2 = VDD (`BAT`) | | not connected |
| pin 3 = GND | ↔ | GND |
| pin 4 = **TX** | ↔ | **RX** |
| pin 5 = **RX** | ↔ | **TX** |

**Serial link parameters:**

- Speed: **1200 baud**
- Format: **8N1** (8 data bits, no parity, 1 stop bit)

⚠️ The WatchDog must be powered by its Li-ion cell (`LI-ION` connector).

<img src="images/Zeus Solar Battery WatchDog - SBMS WatchDog Serial Prog.jpeg" width=500>

### Waking the module up

The module is asleep almost all the time. **Before each command, send a BREAK** from your terminal (about 11 ms or more), wait a few milliseconds, then send the command.

### Commands

- `GET NAME` → `NAME=value`
- `SET NAME value` → `OK` or `ERR`

Tips:

- Send the whole command at once, followed by Enter (CR and/or LF).
- Upper or lower case, as you like.
- Values are whole numbers of up to 4 digits, optionally negative, except `TEMP_PROT` and `VOLT_PROT`, which take `ON` or `OFF`.
- A successful `SET` is **saved immediately** and kept after a power loss.

### Status and information

| Command | What it shows |
|---|---|
| `GET STATUS` | Version, temperature, voltage, state of the outputs |
| `GET INFOS` | Summary of all thresholds |
| `GET T90` | Self-test of the chip's factory temperature calibration: about `90` = OK, `0` = rejected (the internal temperature will always be `ERR`) |

**`GET STATUS`**

```
VER : v10.0.X
TEMP_C : 25
BAT_V : 4.185
BAT_EN : ENABLED
CHRG_EN : ENABLED
TEMP_PROT : ON
VOLT_PROT : ON
PARAMS : OK
```

- `TEMP_C` → internal sensor, with `TEMP_OFFSET_C` applied.
- `BAT_V : 0.000` → voltage measurement problem: charging is switched off for safety (unless `VOLT_PROT` is `OFF`).
- `TEMP_C : ERR` → temperature measurement problem: charging stays allowed under the voltage rules (see [Charging control](#charging-control)), and after a delay the module works without temperature protection (see [Temperature sensor failure](#temperature-sensor-failure)). If it stays in `ERR`, check `GET T90`.
- `BAT_EN` / `CHRG_EN` → `ENABLED` = battery connected / solar panel connected.
- `TEMP_PROT` / `VOLT_PROT` → `ON` = protection active, `OFF` = protection disabled.
- `PARAMS` → where the settings in use come from:

| `PARAMS` | Meaning | What to do |
|---|---|---|
| `OK` | Your settings are saved and in use | Nothing |
| `DEFAULT` | No setting has ever been saved: the default values are in use | Nothing, this is normal if you never used `SET` |
| `ERR` | Your saved settings were lost (damaged, or older format after a firmware update) or could not be saved: the default values are in use | Re-apply your settings with `SET` |

**`GET INFOS`** (default values)

```
BAT_MV : 3200 ; 3400 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-17 ; 0 ; 47 ; 62/65
REBOOT_D : 0
```

- `BAT_MV`: low ; OK ; high voltage thresholds (mV)
- `BAT_S`: low ; OK ; high confirmation durations (s)
- `TEMP_C`: danger low (on/off) ; charging min ; charging max ; danger high (off/on)
- `REBOOT_D`: periodic reboot (days)


### Settings

| Name | Meaning | Unit | Min | Max | Default |
|---|---|---|---|---|---|
| `BAT_LOW_MV` | Low voltage → battery off | mV | 3100 | 3499 | 3200 |
| `BAT_LOW_S` | Delay before battery off | s | 0 | 4318 | 600 |
| `BAT_OK_MV` | Voltage OK → battery on + reset | mV | 3101 | 3500 | 3400 |
| `BAT_OK_S` | Delay before battery on | s | 0 | 4318 | 0 |
| `BAT_HIGH_MV` | Overvoltage → charging off | mV | 3102 | 5500 | 4280 |
| `BAT_HIGH_S` | Delay before charging off / back on | s | 0 | 4318 | 0 |
| `TEMP_LOW_C` | Below → charging off | °C | -40 | 125 | 0 |
| `TEMP_HIGH_C` | Above → charging off | °C | -40 | 125 | 47 |
| `TEMP_DANGER_LOW_C` | At or below → total shutdown | °C | -40 | 125 | -20 |
| `TEMP_DANGER_HIGH_C` | At or above → total shutdown | °C | -40 | 125 | 65 |
| `TEMP_DANGER_HYST_C` | Margin to leave the total shutdown | °C | 0 | 41 (*) | 3 |
| `TEMP_OFFSET_C` | Internal sensor correction (not the DS18B20) | °C | -9 | 9 | -3 |
| `REBOOT_D` | Periodic reboot, 0 = off | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Reset pulse duration (main board unpowered) | ms | 0 | 8191 | 5000 |
| `BAT_CUT_PULSE_MS` | Extra battery cut before the reset (startup, reboot) | ms | 0 | 8191 | 0 |
| `TEMP_PROT` | Temperature protection | — | `OFF` | `ON` | `ON` |
| `VOLT_PROT` | Voltage protection | — | `OFF` | `ON` | `ON` |

(*) With the default danger thresholds.

Why these voltage limits:

- `BAT_LOW_MV` ≥ 3100: protects the cell while it is recharged with the battery off, and keeps the DS18B20 working.
- `BAT_OK_MV` ≤ 3500: see [Charging while the battery is off](#charging-while-the-battery-is-off).
- `RESET_PULSE_MS` = 5000: the main board is unpowered for 5 s, which is enough for a clean restart. Shorten it only after checking that your main board really restarts.
- `BAT_CUT_PULSE_MS` = 0: the reset pulse alone already power-cycles the main board.

The settings must stay in this order, otherwise `SET` answers `ERR`:

- `BAT_LOW_MV` < `BAT_OK_MV` < `BAT_HIGH_MV`
- `TEMP_DANGER_LOW_C` < `TEMP_LOW_C` < `TEMP_HIGH_C` < `TEMP_DANGER_HIGH_C`
- `TEMP_DANGER_LOW_C` + `TEMP_DANGER_HYST_C` + 1 < `TEMP_DANGER_HIGH_C` − `TEMP_DANGER_HYST_C`

To move a whole range, change the values in an order that keeps this rule true at each step (for example, raise `BAT_OK_MV` before `BAT_LOW_MV`).

### Example session

Each command is preceded by a BREAK.

```
> GET STATUS
VER : v10.0.X
TEMP_C : 23
BAT_V : 3.980
BAT_EN : ENABLED
CHRG_EN : ENABLED
TEMP_PROT : ON
VOLT_PROT : ON
PARAMS : OK

> SET BAT_LOW_MV 3300
OK

> GET BAT_LOW_MV
BAT_LOW_MV=3300

> SET BAT_LOW_MV 3000
ERR

> SET BAT_OK_MV 3600
ERR

> SET TEMP_DANGER_HYST_C 2
OK

> GET INFOS
BAT_MV : 3300 ; 3400 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-18 ; 0 ; 47 ; 63/65
REBOOT_D : 0

> SET VOLT_PROT OFF
OK

> GET VOLT_PROT
VOLT_PROT=OFF

> SET VOLT_PROT 1
ERR

> SET VOLT_PROT ON
OK
```

`ERR` means: unknown command or name, invalid number (or not `ON` / `OFF` for `TEMP_PROT` and `VOLT_PROT`), value out of range, settings out of order, or the setting could not be saved.

### Calibrating the temperature

`TEMP_OFFSET_C` corrects the internal sensor only (not the DS18B20). It is still useful with a DS18B20, since the internal sensor takes over when a DS18B20 measurement fails.

Calibrate in UART mode (DS18B20 unplugged at startup). Place a reference thermometer on the PIC chip, compare with `TEMP_C` in `GET STATUS`, and adjust `TEMP_OFFSET_C` by the difference. Do it as close as possible to the most important threshold.

---

## Programming the firmware

The firmware is programmed into a **blank PIC16F13115**.

Use a **Microchip SNAP (PG164100)** or **MPLAB PICkit 5** programmer.

1. Install MPLAB X IDE (it includes MPLAB IPE) and connect the programmer to the PC.
2. Wire it to the WatchDog through the 5-pin ICSP port: pin 1 = MCLR, 2 = VDD, 3 = GND, 4 = PGD, 5 = PGC.
3. In MPLAB IPE, select **PIC16F13115** and your programmer, click **Connect**, then select the `.hex` file.
4. Click **Program** and check that programming and verification both pass.

⚠️ Before programming:

- The WatchDog must be powered by its Li-ion cell.
- Unplug the DS18B20 and its pull-up resistor, and remove the version LED jumper.

After programming, with the USB serial link:

- Run `GET STATUS`:
  - `PARAMS : DEFAULT` → normal: the default values are in use.
  - `TEMP_PROT` and `VOLT_PROT` should both be `ON`.
- Run `GET T90`: about `90` = OK.
- Run `GET INFOS` to check the thresholds, and adjust them with `SET` if needed.

