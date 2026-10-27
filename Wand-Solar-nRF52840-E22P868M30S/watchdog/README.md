# Wand Smart Battery Management System

Battery **voltage** and **temperature** watchdog for the Wand board, based on a PIC16F13114 microcontroller.

Firmware version: **v8.5.X**

<img src="../images/Wand Solar nRF52840 E22P-868M30S - Schematics v8 WatchDog 2.png">

## What it does

This little module keeps an eye on your battery and temperature, and protects them automatically.

About every 17 seconds, it wakes up, checks the battery voltage and the temperature, acts if needed, then goes back to sleep.

Its key feature: it can **completely cut the power** of the Meshcore board (`BAT_EN`) and **reset** it (`RESET_CPU`). This reboots even the most stubborn boards and fully discharges the capacitors for a clean restart.

---

## How it works

### At startup

1. The battery and charging are switched **off**.
2. If the version LED jumper is fitted, the LED blinks the firmware version.
3. After 5 s (`BAT_CUT_PULSE_MS`), the module takes a first measurement.
4. If the temperature and the voltage are OK, the battery is switched **on** and the Meshcore board receives a 1 s reset pulse (`RESET_PULSE_MS`). Otherwise the battery stays off until conditions are back to normal. If the temperature cannot be measured, see [Temperature sensor failure](#temperature-sensor-failure).
   A protection switched off (`TEMP_PROT` or `VOLT_PROT`, see below) is not checked here.

### Battery protection (voltage)

| Condition | Action |
|---|---|
| Voltage below `BAT_LOW_MV` (3.2 V) for `BAT_LOW_S` (10 min) | Battery **off** |
| Voltage above `BAT_OK_MV` (3.5 V) for `BAT_OK_S` (immediate) | Battery **on** + reset pulse |

### Charging control

Charging is **allowed** only when all of these are true:

- the temperature is between `TEMP_LOW_C` and `TEMP_HIGH_C` (0 °C to 47 °C), or the temperature measurement has failed (see below);
- the voltage has not exceeded `BAT_HIGH_MV` (4.28 V) for `BAT_HIGH_S` (immediate);
- the voltage measurement is working.

**Temperature measurement failure:** charging stays **allowed**, so that the battery can always be recharged (for example when the battery is so low that the sensor no longer works: the chip needs at least 2.5 V for it). Charging is then controlled by the voltage rules only: it stops above `BAT_HIGH_MV`. Exception: if a dangerous temperature was measured just before the failure, charging stays off (see [Temperature sensor failure](#temperature-sensor-failure)).

A protection switched off removes its own conditions from this list (see [Disabling a protection](#disabling-a-protection)).

After an overvoltage, charging comes back once the voltage has been back at or below `BAT_HIGH_MV` for `BAT_HIGH_S`.

### Thermal protection

| Condition | Action |
|---|---|
| Temperature at or above `TEMP_DANGER_HIGH_C` (65 °C) | **Total shutdown**: battery and charging off |
| Temperature at or below `TEMP_DANGER_LOW_C` (-20 °C) | **Total shutdown**: battery and charging off |

The total shutdown has **priority over everything else**. It ends when the temperature is back below 62 °C and above -17 °C (`TEMP_DANGER_HYST_C` = 3 °C margin), so the board is not switched on and off repeatedly around a threshold. The battery is then switched back on with a reset pulse, unless the voltage is too low.

### Temperature sensor failure

If the temperature cannot be measured (`TEMP_C : ERR`) for a while, the module continues **as if `TEMP_PROT` were `OFF`**: only the voltage rules apply, and the battery is switched back on with a reset pulse if the voltage allows it.

| Before the failure | Delay before working without temperature |
|---|---|
| Normal temperature, or failure from startup | about 51 s (3 checks) |
| Dangerous temperature measured | about 1 hour, to let the battery cool down |

- Your `TEMP_PROT` setting is **not changed**: as soon as the temperature can be measured again, the temperature protection works again by itself.
- The delays are counted on the regular ~17 s checks only: they are paused while you send commands on the serial link.

### Periodic reboot

Optional: every `REBOOT_D` days, the module cuts the battery for 5 s, switches it back on and resets the Meshcore board. Disabled by default (`REBOOT_D` = 0). The reboot is postponed while the battery is off for protection.

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

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Reset.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram VDD.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Temp.jpg">

### Good to know

- **Timing**: the module checks every ~17 s, so all durations are rounded up to a multiple of 17 s and are approximate. They are never shorter than requested. A value of 0 acts immediately.
- **Power consumption**: very low, the chip sleeps almost all the time.

---

## Version LED

An optional LED (with a 1 kΩ resistor, between PIN4 and GND) can be connected through a **jumper**. At startup it blinks the version.

- Fit the jumper only to check the version, then remove it (otherwise the LED stays lit and drains the battery).
- Do not fit it while a programmer or the USB-serial adapter is connected.

---

## Configuration (serial link)

### Connection

Configuration is done via the module's ICSP port, connected to a USB-to-serial adapter.

| ICSP port | ↔ | USB-serial adapter |
|---|---|---|
| pin 1  | ↔ |  |
| pin 2  | ↔ |  |
| pin 3 = GND | ↔ | GND |
| pin 4 = **TX** | ↔ | **RX** |
| pin 5 = **RX** | ↔ | **TX** |

**Serial link parameters:**

- Speed: **1200 baud**
- Format: **8N1** (8 data bits, no parity, 1 stop bit)

⚠️ The Wand must be powered by its own battery.

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Serial CNX.jpeg" width=500>
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Serial Prog.jpeg" width=500>
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Prog Cnx.jpeg" width=500>

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
| `GET T90` | Self-test of the chip's factory temperature calibration: about `90` = OK, `0` = rejected (the temperature will always be `ERR`) |

**`GET STATUS`**

```
VER : v8.5.4
TEMP_C : 25
BAT_V : 4.185
BAT_EN : ENABLED
CHRG_EN : ENABLED
TEMP_PROT : ON
VOLT_PROT : ON
PARAMS : OK
```

- `BAT_V : 0.000` → voltage measurement problem: charging is switched off for safety (unless `VOLT_PROT` is `OFF`).
- `TEMP_C : ERR` → temperature measurement problem: charging stays allowed under the voltage rules (see [Charging control](#charging-control)), and after a delay the module works without temperature protection (see [Temperature sensor failure](#temperature-sensor-failure)). If it stays in `ERR`, check `GET T90`.
- `TEMP_PROT` / `VOLT_PROT` → `ON` = protection active, `OFF` = protection disabled.
- `PARAMS` → where the settings in use come from:

| `PARAMS` | Meaning | What to do |
|---|---|---|
| `OK` | Your settings are saved and in use | Nothing |
| `DEFAULT` | No setting has ever been saved: the default values are in use | Nothing, this is normal if you never used `SET` |
| `ERR` | Your saved settings were lost (damaged, or older format after a firmware update) or could not be saved: the default values are in use | Re-apply your settings with `SET` |

**`GET INFOS`** (default values)

```
BAT_MV : 3200 ; 3500 ; 4280
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
| `BAT_LOW_MV` | Low voltage → battery off | mV | 1800 | 5500 | 3200 |
| `BAT_LOW_S` | Delay before battery off | s | 0 | 4318 | 600 |
| `BAT_OK_MV` | Voltage OK → battery on + reset | mV | 1800 | 5500 | 3500 |
| `BAT_OK_S` | Delay before battery on | s | 0 | 4318 | 0 |
| `BAT_HIGH_MV` | Overvoltage → charging off | mV | 1800 | 5500 | 4280 |
| `BAT_HIGH_S` | Delay before charging off / back on | s | 0 | 4318 | 0 |
| `TEMP_LOW_C` | Below → charging off | °C | -40 | 125 | 0 |
| `TEMP_HIGH_C` | Above → charging off | °C | -40 | 125 | 47 |
| `TEMP_DANGER_LOW_C` | At or below → total shutdown | °C | -40 | 125 | -20 |
| `TEMP_DANGER_HIGH_C` | At or above → total shutdown | °C | -40 | 125 | 65 |
| `TEMP_DANGER_HYST_C` | Margin to leave the total shutdown | °C | 0 | 41 (*) | 3 |
| `TEMP_OFFSET_C` | Temperature correction | °C | -9 | 9 | -3 |
| `REBOOT_D` | Periodic reboot, 0 = off | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Reset pulse duration | ms | 0 | 8191 | 1000 |
| `BAT_CUT_PULSE_MS` | Battery cut duration (startup, reboot) | ms | 0 | 8191 | 5000 |
| `TEMP_PROT` | Temperature protection | — | `OFF` | `ON` | `ON` |
| `VOLT_PROT` | Voltage protection | — | `OFF` | `ON` | `ON` |

(*) With the default danger thresholds.

The settings must stay in this order, otherwise `SET` answers `ERR`:

- `BAT_LOW_MV` < `BAT_OK_MV` < `BAT_HIGH_MV`
- `TEMP_DANGER_LOW_C` < `TEMP_LOW_C` < `TEMP_HIGH_C` < `TEMP_DANGER_HIGH_C`

To move a whole range, change the values in an order that keeps this rule true at each step (for example, raise `BAT_OK_MV` before `BAT_LOW_MV`).

### Example session

Each command is preceded by a BREAK.

```
> GET STATUS
VER : v8.5.4
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

> SET BAT_LOW_MV 1000
ERR

> SET TEMP_DANGER_HYST_C 2
OK

> GET INFOS
BAT_MV : 3300 ; 3500 ; 4280
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

`TEMP_OFFSET_C` corrects the temperature reading. Place a reference thermometer on the PIC chip, compare with `TEMP_C` in `GET STATUS`, and adjust `TEMP_OFFSET_C` by the difference.

---

## Firmware update

Use a **Microchip SNAP (PG164100)** or **MPLAB PICkit 5** programmer.

1. Install MPLAB X IDE (it includes MPLAB IPE) and connect the programmer to the PC.
2. Wire it to the Wand through the 8-pin header: pin 1 = MCLR, 2 = VDD, 3 = GND, 4 = PGD, 5 = PGC.
3. In MPLAB IPE, select **PIC16F13114** and your programmer, click **Connect**, then select the `.hex` file.
4. Click **Program** and check that programming and verification both pass.

⚠️ Before updating:

- The Wand must be powered by its own battery.
- **Updating from v8.5.3 or earlier to v8.5.4 resets all settings to their default values** (the way settings are saved has changed). If you changed any setting, write down your values first: `GET INFOS`, plus `GET TEMP_OFFSET_C`, `GET TEMP_DANGER_HYST_C`, `GET RESET_PULSE_MS` and `GET BAT_CUT_PULSE_MS`, which `GET INFOS` does not show.

After updating, with USB serial link:

- Run `GET STATUS`:
  - `PARAMS : ERR` → your settings were reset; re-apply them with `SET`.
  - `PARAMS : DEFAULT` → no setting had been saved before; the default values are in use, nothing to do.
  - `TEMP_PROT` and `VOLT_PROT` should both be `ON`.
- Run `GET INFOS` to check your settings.

<img src="../images/Wand Solar nRF52840 E22P-868M30S - MPLAB SNAP.jpg" width=400>
