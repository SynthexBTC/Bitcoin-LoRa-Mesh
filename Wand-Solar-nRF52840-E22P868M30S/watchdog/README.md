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
4. If the temperature and the voltage are OK, the battery is switched **on** and the Meshcore board receives a 1 s reset pulse (`RESET_PULSE_MS`). Otherwise the battery stays off until conditions are back to normal.

### Battery protection (voltage)

| Condition | Action |
|---|---|
| Voltage below `BAT_LOW_MV` (3.2 V) for `BAT_LOW_S` (10 min) | Battery **off** |
| Voltage above `BAT_OK_MV` (3.8 V) for `BAT_OK_S` (immediate) | Battery **on** + reset pulse |

### Charging control

Charging is **allowed** only when all of these are true:

- the temperature is between `TEMP_LOW_C` and `TEMP_HIGH_C` (0 °C to 47 °C);
- the voltage has not exceeded `BAT_HIGH_MV` (4.28 V) for `BAT_HIGH_S` (immediate);
- the voltage and temperature measurements are working.

After an overvoltage, charging comes back once the voltage has been back at or below `BAT_HIGH_MV` for `BAT_HIGH_S`.

### Thermal protection

| Condition | Action |
|---|---|
| Temperature at or above `TEMP_DANGER_HIGH_C` (65 °C) | **Total shutdown**: battery and charging off |
| Temperature at or below `TEMP_DANGER_LOW_C` (-20 °C) | **Total shutdown**: battery and charging off |

The total shutdown has **priority over everything else**. It ends when the temperature is back below 62 °C and above -17 °C (`TEMP_DANGER_HYST_C` = 3 °C margin), so the board is not switched on and off repeatedly around a threshold. The battery is then switched back on with a reset pulse, unless the voltage is too low.

### Periodic reboot

Optional: every `REBOOT_D` days, the module cuts the battery for 5 s, switches it back on and resets the Meshcore board. Disabled by default (`REBOOT_D` = 0). The reboot is postponed while the battery is off for protection.

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Reset.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram VDD.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Temp.jpg">

### Good to know

- **Timing**: the module checks every ~17 s, so all durations are rounded up to a multiple of 17 s and are approximate. They are never shorter than requested. A value of 0 acts immediately.
- **Temperature**: the module measures the temperature of its own chip, not of the battery cell, with an accuracy of about ±5 °C. It does not replace a dedicated thermal protection of the battery.
- **Power consumption**: very low, the chip sleeps almost all the time.

---

## Version LED

An optional LED (with a 1 kΩ resistor, between ICSPDAT and GND) can be connected through a **jumper**. At startup it blinks the version.

- Fit the jumper only to check the version, then remove it (otherwise the LED stays lit and drains the battery).
- Do not fit it while a programmer or the USB-serial adapter is connected.

---

## Configuration (serial link)

### Connection

Connect a **3.3 V** USB-serial adapter to the module's ICSP port:

| ICSP port | ↔ | USB-serial adapter |
|---|---|---|
| pin 3 = GND | ↔ | GND |
| pin 4 = **TX** | ↔ | **RX** |
| pin 5 = **RX** | ↔ | **TX** |

Settings: **1200 baud, 8N1**.

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
- Values are whole numbers of up to 4 digits, optionally negative.
- A successful `SET` is **saved immediately** and kept after a power loss.

### Status and information

| Command | What it shows |
|---|---|
| `GET STATUS` | Version, temperature, voltage, state of the outputs |
| `GET INFOS` | Summary of all thresholds |
| `GET T90` | Temperature sensor self-test (should be about 90) |

**`GET STATUS`**

```
VER : v8.5.3
TEMP_C : 25
BAT_V : 4.185
BAT_EN : ENABLED
CHRG_EN : ENABLED
PARAMS : OK
```

- `TEMP_C : ERR` or `BAT_V : 0.000` → measurement problem (charging is switched off for safety).
- `PARAMS : ERR` → your settings are not saved and the default values are in use. Re-apply your settings with `SET`.

**`GET INFOS`** (default values)

```
BAT_MV : 3200 ; 3800 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-17 ; 0 ; 47 ; 62/65
REBOOT_D : 0
```

- `BAT_MV`: low ; OK ; high voltage thresholds (mV)
- `BAT_S`: low ; OK ; high confirmation durations (s)
- `TEMP_C`: danger low (on/off) ; charging min ; charging max ; danger high (off/on)
- `REBOOT_D`: periodic reboot (days)

**`GET T90`**: `T90=90` (80 to 100) means the temperature sensor is OK. Any other value means it is faulty: the battery then stays off for safety.

### Settings

| Name | Meaning | Unit | Min | Max | Default |
|---|---|---|---|---|---|
| `BAT_LOW_MV` | Low voltage → battery off | mV | 1800 | 5500 | 3200 |
| `BAT_LOW_S` | Delay before battery off | s | 0 | 4318 | 600 |
| `BAT_OK_MV` | Voltage OK → battery on + reset | mV | 1800 | 5500 | 3800 |
| `BAT_OK_S` | Delay before battery on | s | 0 | 4318 | 0 |
| `BAT_HIGH_MV` | Overvoltage → charging off | mV | 1800 | 5500 | 4280 |
| `BAT_HIGH_S` | Delay before charging off / back on | s | 0 | 4318 | 0 |
| `TEMP_LOW_C` | Below → charging off | °C | -40 | 125 | 0 |
| `TEMP_HIGH_C` | Above → charging off | °C | -40 | 125 | 47 |
| `TEMP_DANGER_LOW_C` | At or below → total shutdown | °C | -40 | 125 | -20 |
| `TEMP_DANGER_HIGH_C` | At or above → total shutdown | °C | -40 | 125 | 65 |
| `TEMP_DANGER_HYST_C` | Margin to leave the total shutdown | °C | 0 | 41 (*) | 3 |
| `TEMP_OFFSET_C` | Temperature correction | °C | -8192 | 8191 | -2 |
| `REBOOT_D` | Periodic reboot, 0 = off | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Reset pulse duration | ms | 0 | 8191 | 1000 |
| `BAT_CUT_PULSE_MS` | Battery cut duration (startup, reboot) | ms | 0 | 8191 | 5000 |

(*) With the default danger thresholds.

The settings must stay in this order, otherwise `SET` answers `ERR`:

- `BAT_LOW_MV` < `BAT_OK_MV` < `BAT_HIGH_MV`
- `TEMP_DANGER_LOW_C` < `TEMP_LOW_C` < `TEMP_HIGH_C` < `TEMP_DANGER_HIGH_C`

To move a whole range, change the values in an order that keeps this rule true at each step (for example, raise `BAT_OK_MV` before `BAT_LOW_MV`).

### Example session

Each command is preceded by a BREAK.

```
> GET STATUS
VER : v8.5.3
TEMP_C : 23
BAT_V : 3.980
BAT_EN : ENABLED
CHRG_EN : ENABLED
PARAMS : OK

> SET BAT_LOW_MV 3300
OK

> GET BAT_LOW_MV
BAT_LOW_MV=3300

> SET BAT_LOW_MV 1000
ERR

> SET TEMP_DANGER_HYST_C 5
OK

> GET INFOS
BAT_MV : 3300 ; 3800 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-15 ; 0 ; 47 ; 60/65
REBOOT_D : 0
```

`ERR` means: unknown command or name, invalid number, value out of range, or settings out of order.

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

After updating, with USB serial link:

- Run `GET STATUS`: if `PARAMS : ERR`, your settings were reset; re-apply them with `SET`.
- Run `GET INFOS` to check your settings.
- Run `GET T90` (should be about 90).

<img src="../images/Wand Solar nRF52840 E22P-868M30S - MPLAB SNAP.jpg" width=400>
