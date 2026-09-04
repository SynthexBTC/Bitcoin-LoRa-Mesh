# Wand Smart Battery Management System

Battery **voltage** and **temperature** watchdog based on a PIC16F13114 microcontroller.

<img src="../images/Wand Solar nRF52840 E22P-868M30S - Schematics v8 WatchDog.png">

## What it does

This little module keeps an eye on your power and temperature, and takes care of your battery automatically.  

Every 16 seconds, it wakes up, takes a quick look at the power level and the temperature, then goes back to sleep.

### Operating Logic

- **At startup**: `BAT_EN` is set to the "battery disabled" state, a pulse is sent on `RESET_CPU`, then the module goes into permanent sleep (WDT wake-up every 16 s).
- **On each wake-up**, the module measures VDD and temperature, then applies the following rules:
  - If VDD stays **below the low threshold** `BAT_LOW_MV` for the confirmation duration `BAT_LOW_S` (10 min by default) → the battery is **disabled** (`BAT_EN` cut off).
  - If VDD exceeds the **re-enable threshold** `BAT_OK_MV` (after confirmation `BAT_OK_S`) → the battery is **re-enabled** (`BAT_EN`), followed by a pulse on `RESET_CPU`.
  - If VDD exceeds the **overvoltage threshold** `BAT_HIGH_MV` for the confirmation duration `BAT_HIGH_S` → **charging is disabled** (`CHRG_EN`), independently of the temperature-based charging window below. It is **re-enabled** only once VDD has been back at or below `BAT_HIGH_MV` for `BAT_HIGH_S` as well (symmetric confirmation on both edges of this threshold — no instant hysteresis).
  - If temperature goes outside the range `[TEMP_LOW_C ; TEMP_HIGH_C]` (0 °C to 47 °C by default) → **charging is disabled** (`CHRG_EN`); within the normal range, charging is allowed **unless** the overvoltage cutoff above is also active (`CHRG_EN` follows the logical AND of "temperature in range" and "no overvoltage cutoff").
  - If temperature exceeds `TEMP_DANGER_HIGH_C` (65 °C by default) or drops below `TEMP_DANGER_LOW_C` (-20 °C by default) → **total safety shutdown** (both battery and charging disabled), with absolute priority over any other logic.
  - When temperature returns to the normal range after a thermal emergency, the battery is **re-enabled immediately** (without waiting for the `BAT_OK_MV`/`BAT_OK_S` confirmation), provided VDD is not below the low threshold.

⚠️ **Note on timing accuracy**: the module only wakes up every 16 seconds (no real-time clock). Confirmation durations (`BAT_LOW_S`, `BAT_OK_S`, `BAT_HIGH_S`) are therefore always **rounded up to the next multiple of 16 s**. For example, an entered value between 1 and 16 s results in an actual delay of 16 s, between 17 and 32 s an actual delay of 32 s, and so on. The value stored and read back via `GET` remains exactly the one entered; only the physically observed delay is quantized in 16 s steps (always rounded up, so never less protective than requested — at worst 15 s more).

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram VDD.jpg">

<img src="../images/Wand Solar nRF52840 E22P-868M30S - SBMS Watchdog Diagram Temp.jpg">

## Power consumption

About half a microamp on average, low enough to run for years on a battery.

---

## Connecting for Configuration (Serial Link)

Configuration is done via the module's ICSP port, connected to a USB-to-serial adapter.

| ICSP Port | ↔ | USB-Serial Adapter |
|---|---|---|
| pin 3 = GND | ↔ | GND |
| pin 4 = ICSPDAT = TX | ↔ | RX |
| pin 5 = ICSPCLK = RX | ↔ | TX |

**Serial link parameters:**

- Speed: **1200 baud**
- Format: **8N1** (8 data bits, no parity, 1 stop bit)

---

## Command Syntax

Two commands are available:

- `GET NAME` → response `NAME=value` or `ERR` (or, for `GET INFOS`, a multi-line response — see below)
- `SET NAME value` → response `OK` or `ERR`

**Important: a successful `SET` immediately saves all parameters to Flash memory** (no separate save command needed).

Commands and parameter names are case-insensitive.

---

## Read-Only Commands (Not Adjustable)

| Command | Description | Response |
|---|---|---|
| `GET TEMP` or `GET TEMPERATURE` | Current temperature | in °C |
| `GET VOLTAGE` or `GET MV` | Current VDD voltage | in mV (0 = measurement failure) |
| `GET VERSION` or `GET VER` | Firmware version | version string |
| `GET INFOS` | Summary of the main thresholds, grouped by topic | 4 lines — see format below |

### `GET INFOS` response format

One line per group, current values separated by ` ; `:

```
BAT_MV : BAT_LOW_MV ; BAT_OK_MV ; BAT_HIGH_MV
BAT_S : BAT_LOW_S ; BAT_OK_S ; BAT_HIGH_S
TEMP_C : TEMP_DANGER_LOW_C ; TEMP_LOW_C ; TEMP_HIGH_C ; TEMP_DANGER_HIGH_C
REBOOT_D : REBOOT_D
```

---

## Adjustable Parameters (GET/SET)

| Name | Meaning | Unit | Min | Max | Default |
|---|---|---|---|---|---|
| `BAT_LOW_MV` | VDD low threshold → disables `BAT_EN` (after confirmation `BAT_LOW_S`) | mV | 0 | 8191 | 3300 |
| `BAT_OK_MV` | VDD re-enable threshold → re-enables `BAT_EN` + `RESET_CPU` pulse (after confirmation `BAT_OK_S`) | mV | 0 | 8191 | 3800 |
| `BAT_LOW_S` | Confirmation duration before battery shutdown — 16 s resolution | s | 0 | 4080 | 600 |
| `BAT_OK_S` | Confirmation duration before re-enabling `BAT_EN` — 16 s resolution | s | 0 | 4080 | 0 |
| `BAT_HIGH_MV` | VDD overvoltage threshold → disables `CHRG_EN` (charging), independently of `TEMP_LOW_C`/`TEMP_HIGH_C` below (after confirmation `BAT_HIGH_S`) | mV | 0 | 8191 | 4280 |
| `BAT_HIGH_S` | Confirmation duration, applied both before disabling and before re-enabling `CHRG_EN` on the `BAT_HIGH_MV` threshold (symmetric) — 16 s resolution | s | 0 | 4080 | 0 |
| `TEMP_LOW_C` | Below this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 0 |
| `TEMP_HIGH_C` | Above this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 47 |
| `TEMP_DANGER_HIGH_C` | Above this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | 65 |
| `TEMP_DANGER_LOW_C` | Below this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | -20 |
| `TEMP_OFFSET_C` | Additive offset applied to the temperature reading (measured_c + offset) | °C | -8192 | 8191 | -2 |
| `REBOOT_D` | Automatic reboot period (battery cut/re-enable + reset pulse); 0 = disabled | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Duration of the `RESET_CPU` pulse | ms | 0 | 8191 | 1000 |
| `BAT_CUT_PULSE_MS` | Duration of `BAT_EN` cutoff (at startup and before each periodic reboot) | ms | 0 | 8191 | 5000 |

---

## Example Session

```
> GET VERSION
VERSION=v8.2.6

> GET TEMP
TEMP=23

> GET VOLTAGE
VOLTAGE=3980

> SET BAT_LOW_MV 3200
OK

> GET BAT_LOW_MV
BAT_LOW_MV=3200

> SET BAT_LOW_MV 99999
ERR

> GET INFOS
BAT_MV : 3200 ; 3800 ; 4280
BAT_S : 600 ; 0 ; 0
TEMP_C : -20 ; 0 ; 47 ; 65
REBOOT_D : 0
```

(`ERR` is returned if the parameter name is unknown, if the value is not a valid integer, or if it is outside the Min/Max bounds listed above.)

---

## Points of Attention for the User

- A `VOLTAGE`/`MV` value of **0** indicates a **voltage measurement failure**, not an actual zero voltage (physically impossible during normal operation).
- The total safety shutdown (high or low thermal emergency) takes **absolute priority over all VDD/charging logic**: it applies even when voltage conditions would normally call for re-enabling the battery.
- `BAT_OK_MV`/`BAT_OK_S` control **re-enabling the battery** (`BAT_EN`); `BAT_HIGH_MV`/`BAT_HIGH_S` are a **separate, independent** overvoltage cutoff on **charging** (`CHRG_EN`) only — the two thresholds are not related and can be set independently.
- The `TEMP_OFFSET_C` offset defaults to **-2 °C** (drift compensation); any value read via `GET TEMP` already includes this offset.
- Any change made via `SET` is **saved immediately and automatically** to Flash memory — it is therefore retained after a power loss.

---
