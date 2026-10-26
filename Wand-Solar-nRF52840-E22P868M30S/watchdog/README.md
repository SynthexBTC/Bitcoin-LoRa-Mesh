# Wand Smart Battery Management System

Battery **voltage** and **temperature** watchdog based on a PIC16F13114 microcontroller.

Firmware version: **v8.5.X**

<img src="../images/Wand Solar nRF52840 E22P-868M30S - Schematics v8 WatchDog 2.png">

## What it does

This little module keeps an eye on your power and temperature, and takes care of your battery automatically.  

About every 17 seconds, it wakes up, takes a quick look at the power level and the temperature, then goes back to sleep.

## Operating Logic

- **At startup**:
  1. `BAT_EN` is set to the "battery disabled" state and `CHRG_EN` to "charging disabled".
  2. If the version LED jumper is fitted, the LED blinks the firmware version (see [Version LED](#version-led)).
  3. After `BAT_CUT_PULSE_MS` (5 s by default), the battery is re-enabled and a pulse of `RESET_PULSE_MS` (1 s by default) is sent on `RESET_CPU`.
  4. A first measurement runs **immediately**, so all protections (temperature, overvoltage, charging window) are active from the start.
  5. The module then goes into permanent sleep, woken up by the watchdog timer about every 17 s, or by the serial link.
- **On each wake-up**, the module measures VDD and temperature, then applies the following rules:
  - If VDD stays **below the low threshold** `BAT_LOW_MV` for the confirmation duration `BAT_LOW_S` (10 min by default) → the battery is **disabled** (`BAT_EN` cut off).
  - If VDD exceeds the **re-enable threshold** `BAT_OK_MV` (after confirmation `BAT_OK_S`) → the battery is **re-enabled** (`BAT_EN`), followed by a pulse on `RESET_CPU`.
  - If VDD exceeds the **overvoltage threshold** `BAT_HIGH_MV` for the confirmation duration `BAT_HIGH_S` → **charging is disabled** (`CHRG_EN`), independently of the temperature-based charging window below. It is **re-enabled** only once VDD has been back at or below `BAT_HIGH_MV` for `BAT_HIGH_S` as well (symmetric confirmation on both edges of this threshold).
  - If temperature goes outside the range `[TEMP_LOW_C ; TEMP_HIGH_C]` (0 °C to 47 °C by default) → **charging is disabled** (`CHRG_EN`). Within that range, charging is allowed **unless** the overvoltage cutoff above is active or the VDD measurement has failed. `CHRG_EN` follows the logical AND of "temperature in range", "no overvoltage cutoff" and "valid VDD measurement".
  - If temperature reaches `TEMP_DANGER_HIGH_C` (65 °C by default) or drops to `TEMP_DANGER_LOW_C` (-20 °C by default) → **total safety shutdown** (both battery and charging disabled), with absolute priority over any other logic.
  - The thermal emergency ends only once the temperature has moved back by `TEMP_DANGER_HYST_C` (3 °C by default): **below 62 °C** and **above -17 °C** with the default values. This hysteresis prevents the battery (and the board, through the `RESET_CPU` pulse) from being switched on and off repeatedly when the temperature hovers around a danger threshold.
  - When the thermal emergency ends:
    - if the battery had **not** been cut for low voltage before the emergency, and VDD is not below `BAT_LOW_MV`, the battery is **re-enabled immediately** (without waiting for `BAT_OK_MV`/`BAT_OK_S`), followed by a `RESET_CPU` pulse;
    - otherwise the battery stays disabled until the normal `BAT_OK_MV`/`BAT_OK_S` re-enable condition is met.

- **Periodic Reboot**: automatic reboot every `REBOOT_D` days (battery cutoff/re-enable + nRF52840 reset pulse). It is postponed while the battery is deliberately disabled (low voltage or thermal emergency).

- **The key feature lies in the management of the reset (RESET_BTN) and, above all, the complete shutdown (BAT_EN) of the Meshcore board. This makes it possible to reboot even the most stubborn boards and fully discharge the capacitors to ensure a clean restart.**

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Reset.jpg">
  
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram VDD.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Temp.jpg">

⚠️ **Note on timing accuracy**: the module only wakes up about every 17 seconds (no real-time clock). The watchdog period is labelled "16 s nominal" in the PIC datasheet, but it runs from the internal 31 kHz low-frequency oscillator: 2^19 / 31 kHz ≈ 16.9 s, so the firmware counts 17 s per wake-up. The datasheet gives no min/max tolerance for this oscillator, so all durations below are approximate.

- Confirmation durations (`BAT_LOW_S`, `BAT_OK_S`, `BAT_HIGH_S`) are counted from the **first wake-up that sees the condition**, and **rounded up to the next multiple of 17 s**. For example, a value between 1 and 17 s gives 17 s after that first detection, and a value between 18 and 34 s gives 34 s.
- Since the condition may have started up to 17 s before it was first seen, the delay from the real onset lies between the rounded value and the rounded value + 17 s. **It is never shorter than requested.** A value of 0 acts at the first detection.
- The value stored and read back via `GET` remains exactly the one entered.
- Confirmation durations and the `REBOOT_D` count only advance on watchdog wake-ups. Under continuous serial traffic (a break more often than every ~17 s), they are paused. The thermal shutdown and the thresholds set to 0 s still act on every wake-up.

## Power consumption

In sleep, the PIC16F13114 datasheet gives **< 900 nA typical** (3 V, 25 °C, watchdog enabled). The short wake-ups every ~17 s and the external circuit (pull-up resistor on `RESET_CPU`, loads on `BAT_EN`/`CHRG_EN`) add to this. The resulting average has **not been measured**.

The version LED must be removed (jumper open) in normal operation: once the serial link is running, ICSPDAT stays high and the LED would draw about 1-2 mA continuously.

---

## Version LED

An optional LED, in series with a **1 kΩ** resistor between ICSPDAT and GND, can be connected through a **jumper**. At startup, before the serial link is enabled, it blinks the firmware version digit: **v8.5.0 → 3 flashes** (100 ms on / 200 ms off).

- Fit the jumper only to check the version, then remove it.
- Do not fit it while a programmer (SNAP / PICkit) or the USB-serial adapter is connected.

---

## Connecting for Configuration (Serial Link)

Configuration is done via the module's ICSP port, connected to a USB-to-serial adapter.

| ICSP Port | ↔ | USB-Serial Adapter |
|---|---|---|
| pin 1  | ↔ |  |
| pin 2  | ↔ |  |
| pin 3 = GND | ↔ | GND |
| pin 4 = ICSPDAT = **TX** | ↔ | **RX** |
| pin 5 = ICSPCLK = **RX** | ↔ | **TX** |

**Serial link parameters:**

- Speed: **1200 baud**
- Format: **8N1** (8 data bits, no parity, 1 stop bit)

⚠️ **Note** : **the Wand must be powered by its own battery to be configured.**  

⚠️ **Waking the module up (BREAK required)**: the module sleeps almost all the time and can only be woken up by a **BREAK** on its RX line.

- Before each command, send a **BREAK** from your terminal: at least 10 bit times, **13 bit times recommended (~11 ms at 1200 baud)**.
- Wait a few milliseconds, then send the command.
- If a normal character is sent first, it is consumed as the wake-up event and the command is received corrupted.

The RX line has an internal pull-up, so it does not float when the adapter is disconnected.
  
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Serial CNX.jpeg" width=500>
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Serial Prog.jpeg" width=500>
<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS WatchDog Prog Cnx.jpeg" width=500>

---

## Command Syntax

Two commands are available:

- `GET NAME` → response `NAME=value` or `ERR` (or, for `GET INFOS` / `GET STATUS`, a multi-line response, see below)
- `SET NAME value` → response `OK` or `ERR`

⚠️ **Note** :

- Configure your terminal to send the whole command at once, followed by an end-of-line character (NL and/or CR).
- Do not leave more than 0.5 s between two characters of a command, otherwise the command is abandoned.
  
**Important: a successful `SET` immediately saves all parameters to Flash memory** (no separate save command needed). If the Flash write fails, the response is `ERR` and the previous value is kept.

Commands and parameter names are case-insensitive. Values are integers of **at most 4 digits**, optionally preceded by `-`.

---

## Read-Only Commands (Not Adjustable)

| Command | Description | Response |
|---|---|---|
| `GET TEMP` | Current temperature | in °C |
| `GET MV` | Current VDD voltage | in mV (0 = measurement failure) |
| `GET VER` | Firmware version | version string |
| `GET INFOS` | Summary of the main thresholds, grouped by topic | 4 lines — see format below |
| `GET STATUS` | Current status of sensors and outputs | 4 lines — see format below |

### `GET INFOS` response format

One line per group, current values separated by ` ; `. The two danger thresholds are shown as **entry/exit** pairs (exit = with the `TEMP_DANGER_HYST_C` hysteresis):

```
BAT_MV : BAT_LOW_MV ; BAT_OK_MV ; BAT_HIGH_MV
BAT_S : BAT_LOW_S ; BAT_OK_S ; BAT_HIGH_S
TEMP_C : TEMP_DANGER_LOW_C/(TEMP_DANGER_LOW_C + TEMP_DANGER_HYST_C) ; TEMP_LOW_C ; TEMP_HIGH_C ; (TEMP_DANGER_HIGH_C - TEMP_DANGER_HYST_C)/TEMP_DANGER_HIGH_C
REBOOT_D : REBOOT_D
```

With the default values:

```
BAT_MV : 3200 ; 3800 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-17 ; 0 ; 47 ; 62/65
REBOOT_D : 0
```

### `GET STATUS` response format

```
TEMP_C : 25
BAT_V : 4.185
BAT_EN : ENABLED
CHRG_EN : ENABLED
```

`TEMP_C` and `BAT_V` are fresh measurements (`BAT_V : 0.000` = measurement failure). `BAT_EN` and `CHRG_EN` show the output states set at the previous wake-up; a new measurement cycle runs right after each command.

---

## Adjustable Parameters (GET/SET)

| Name | Meaning | Unit | Min | Max | Default |
|---|---|---|---|---|---|
| `BAT_LOW_MV` | VDD low threshold → disables `BAT_EN` (after confirmation `BAT_LOW_S`) | mV | 0 | 8191 | 3200 |
| `BAT_OK_MV` | VDD re-enable threshold → re-enables `BAT_EN` + `RESET_CPU` pulse (after confirmation `BAT_OK_S`) | mV | 0 | 8191 | 3800 |
| `BAT_LOW_S` | Confirmation duration before battery shutdown, 17 s resolution | s | 0 | 4318 | 600 |
| `BAT_OK_S` | Confirmation duration before re-enabling `BAT_EN`, 17 s resolution | s | 0 | 4318 | 0 |
| `BAT_HIGH_MV` | VDD overvoltage threshold → disables `CHRG_EN` (charging), independently of `TEMP_LOW_C`/`TEMP_HIGH_C` below (after confirmation `BAT_HIGH_S`) | mV | 0 | 8191 | 4280 |
| `BAT_HIGH_S` | Confirmation duration, applied both before disabling and before re-enabling `CHRG_EN` on the `BAT_HIGH_MV` threshold (symmetric), 17 s resolution | s | 0 | 4318 | 0 |
| `TEMP_LOW_C` | Below this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 0 |
| `TEMP_HIGH_C` | Above this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 47 |
| `TEMP_DANGER_HIGH_C` | At or above this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | 65 |
| `TEMP_DANGER_LOW_C` | At or below this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | -20 |
| `TEMP_DANGER_HYST_C` | Hysteresis to leave the thermal emergency: exit below `TEMP_DANGER_HIGH_C` − hyst and above `TEMP_DANGER_LOW_C` + hyst | °C | 0 | 8191 (*) | 3 |
| `TEMP_OFFSET_C` | Additive offset applied to the temperature reading (measured_c + offset) | °C | -8192 | 8191 | -2 |
| `REBOOT_D` | Automatic reboot period (battery cut/re-enable + reset pulse); 0 = disabled | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Duration of the `RESET_CPU` pulse | ms | 0 | 8191 | 1000 |
| `BAT_CUT_PULSE_MS` | Duration of `BAT_EN` cutoff (at startup and before each periodic reboot) | ms | 0 | 8191 | 5000 |

(*) In practice limited by the consistency rule below: 41 °C at most with the default danger thresholds.

---

## Example Session

Each command below is preceded by a BREAK (see [Waking the module up](#connecting-for-configuration-serial-link)).

```
> GET VERSION
VERSION=v8.5.0

> GET TEMP
TEMP=23

> GET VOLTAGE
VOLTAGE=3980

> SET BAT_LOW_MV 3300
OK

> GET BAT_LOW_MV
BAT_LOW_MV=3300

> SET BAT_LOW_MV 99999
ERR

> SET TEMP_DANGER_HYST_C 5
OK

> GET INFOS
BAT_MV : 3300 ; 3800 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20/-15 ; 0 ; 47 ; 60/65
REBOOT_D : 0
```

`ERR` is returned in these cases:

- the parameter name is unknown;
- the value is not a valid integer of at most 4 digits;
- the value is outside the Min/Max bounds listed above;
- the value breaks a consistency rule;
- the Flash write failed.

---

## Points of Attention for the User

- **Parameter consistency** : a SET operation that violates one of these rules returns ERR, and the value remains unchanged :  
 BAT_LOW_MV < BAT_OK_MV < BAT_HIGH_MV  
 TEMP_DANGER_LOW_C < TEMP_LOW_C < TEMP_HIGH_C < TEMP_DANGER_HIGH_C  
 TEMP_DANGER_LOW_C + TEMP_DANGER_HYST_C + 1 < TEMP_DANGER_HIGH_C − TEMP_DANGER_HYST_C (at least one whole degree to leave the thermal emergency)  
 BAT_CUT_PULSE_MS + RESET_PULSE_MS ≤ 12000  
 To move a whole range, change the parameters in an order that keeps the rules true at every step (e.g. raise `BAT_OK_MV` before raising `BAT_LOW_MV` above its old value).

- A `VOLTAGE`/`MV` value of **0** indicates a **voltage measurement failure**, not an actual zero voltage (physically impossible during normal operation). While the measurement fails, **charging is kept disabled**.
- The total safety shutdown (high or low thermal emergency) takes **absolute priority over all VDD/charging logic**: it applies even when voltage conditions would normally call for re-enabling the battery.
- `BAT_OK_MV`/`BAT_OK_S` control **re-enabling the battery** (`BAT_EN`); `BAT_HIGH_MV`/`BAT_HIGH_S` are a **separate, independent** overvoltage cutoff on **charging** (`CHRG_EN`) only, the two thresholds are not related and can be set independently.
- The `TEMP_OFFSET_C` offset defaults to **-2 °C** (drift compensation); any value read via `GET TEMP` already includes this offset. It is the temperature of the PIC chip itself, measured by its internal sensor.
- Any change made via `SET` is **saved immediately and automatically** to Flash memory, it is therefore retained after a power loss.
- Any reset of the module (power-up, programming, internal watchdog reset) replays the full startup sequence, **including the `BAT_EN` cutoff and the `RESET_CPU` pulse** of the Meshcore board.

---

## Firmware Update

You can update the PIC16F13114 Firmware with a **Microchip SNAP (PG164100)** Programmer, or a MPLAB PICkit 5 Programmer.  

1. Install MPLAB X IDE (it includes MPLAB IPE), connect the SNAP to the PC via USB, and wire it to the powered target through the 8-pin header (pin 1 = MCLR/VPP, 2 = VDD, 3 = GND, 4 = PGD, 5 = PGC).
2. Open MPLAB IPE, select the PIC model (16F13114) and the Snap as the tool, click Connect, then browse to the last `.hex` file.
3. Click Program and confirm that the programming and verify steps both pass.

⚠️ Notes :

- The Wand must be powered by its own battery to be updated with SNAP.
- Remove the version LED jumper before programming.
- **Updating to v8.5.0 resets all parameters to their default values** (the parameter storage format changed). Re-apply your own settings with `SET` after the update, and check them with `GET INFOS`.
  
<img src="../images/Wand Solar nRF52840 E22P-868M30S - MPLAB SNAP.jpg" width=400>
