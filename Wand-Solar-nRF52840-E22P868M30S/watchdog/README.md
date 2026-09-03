# Wand WatchDog

Battery **voltage** and **temperature** watchdog based on a PIC16F13113 microcontroller.

<img src="../images/Wand Solar nRF52840 E22P-868M30S - Schematics v8 WatchDog.png">

## What it does

This little module keeps an eye on your power and temperature, and takes care of your battery automatically.  

Every 32 seconds, it wakes up, takes a quick look at the power level and the temperature, then goes back to sleep.

### Operating Logic

- **At startup**: `BAT_EN` is set to the "battery disabled" state, a pulse is sent on `RESET_CPU`, then the module goes into permanent sleep (WDT wake-up every 32 s).
- **On each wake-up**, the module measures VDD and temperature, then applies the following rules:
  - If VDD stays **below the low threshold** for the confirmation duration `BAT_SEUIL_BAS_S` (10 min by default) → the battery is **disabled** (`BAT_EN` cut off).
  - If VDD exceeds the **high threshold** (after confirmation `BAT_SEUIL_HAUT_S`) → the battery is **re-enabled** (`BAT_EN`), followed by a pulse on `RESET_CPU`.
  - If temperature goes outside the range `[TEMP_SEUIL_BAS_C ; TEMP_SEUIL_HAUT_C]` (0 °C to 47 °C by default) → **charging is disabled** (`CHRG_EN`); within the normal range, charging is allowed.
  - If temperature exceeds `TEMP_SEUIL_URGENCE_HAUT_C` (65 °C by default) or drops below `TEMP_SEUIL_URGENCE_BAS_C` (-20 °C by default) → **total safety shutdown** (both battery and charging disabled), with absolute priority over any other logic.
  - When temperature returns to the normal range after a thermal emergency, the battery is **re-enabled immediately** (without waiting for the `BAT_SEUIL_HAUT_S` delay), provided VDD is not below the low threshold.

⚠️ **Note on timing accuracy**: the module only wakes up every 32 seconds (no real-time clock). Confirmation durations (`BAT_SEUIL_BAS_S`, `BAT_SEUIL_HAUT_S`) are therefore always **rounded up to the next multiple of 32 s**. For example, an entered value between 1 and 32 s results in an actual delay of 32 s, between 33 and 64 s an actual delay of 64 s, and so on. The value stored and read back via `GET` remains exactly the one entered; only the physically observed delay is quantized in 32 s steps (always rounded up, so never less protective than requested).

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

- `GET NAME` → response `NAME=value` or `ERR`
- `SET NAME value` → response `OK` or `ERR`

**Important: a successful `SET` immediately saves all parameters to Flash memory** (no separate save command needed).

Commands and parameter names are case-insensitive.

---

## Read-Only Commands (Not Adjustable)

| Command | Description | Response |
|---|---|---|
| `GET TEMP` or `GET TEMPERATURE` | Current temperature | in °C |
| `GET VOLTAGE` or `GET MV` | Current VDD voltage | in mV (**0 = measurement failure**) |
| `GET VERSION` or `GET VER` | Firmware version | version string |

---

## Adjustable Parameters (GET/SET)

| Name | Meaning | Unit | Min | Max | Default |
|---|---|---|---|---|---|
| `BAT_SEUIL_BAS_MV` | VDD low threshold → disables `BAT_EN` (after confirmation `BAT_SEUIL_BAS_S`) | mV | 0 | 8191 | 3300 |
| `BAT_SEUIL_HAUT_MV` | VDD high threshold → re-enables `BAT_EN` + `RESET_CPU` pulse (after confirmation `BAT_SEUIL_HAUT_S`) | mV | 0 | 8191 | 3800 |
| `BAT_SEUIL_BAS_S` | Confirmation duration before battery shutdown — 32 s resolution | s | 0 | 8160 | 600 |
| `BAT_SEUIL_HAUT_S` | Confirmation duration before re-enable — 32 s resolution | s | 0 | 8160 | 0 |
| `TEMP_SEUIL_BAS_C` | Below this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 0 |
| `TEMP_SEUIL_HAUT_C` | Above this threshold → `CHRG_EN` disabled | °C | -8192 | 8191 | 47 |
| `TEMP_SEUIL_URGENCE_HAUT_C` | Above this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | 65 |
| `TEMP_SEUIL_URGENCE_BAS_C` | Below this threshold → total shutdown (`BAT_EN` + `CHRG_EN`) | °C | -8192 | 8191 | -20 |
| `TEMP_OFFSET_C` | Additive offset applied to the temperature reading (measured_c + offset) | °C | -8192 | 8191 | -2 |
| `REBOOT_J` | Automatic reboot period (battery cut/re-enable + reset pulse); 0 = disabled | days | 0 | 8191 | 0 |
| `RESET_PULSE_MS` | Duration of the `RESET_CPU` pulse | ms | 0 | 8191 | 1000 |
| `BAT_CUT_PULSE_MS` | Duration of `BAT_EN` cutoff (at startup and before each periodic reboot) | ms | 0 | 8191 | 5000 |

---

## Example Session

```
> GET VERSION
VERSION=v1.3

> GET TEMP
TEMP=23

> GET VOLTAGE
VOLTAGE=3980

> SET BAT_SEUIL_BAS_MV 3200
OK

> GET BAT_SEUIL_BAS_MV
BAT_SEUIL_BAS_MV=3200

> SET BAT_SEUIL_BAS_MV 99999
ERR
```

(`ERR` is returned if the parameter name is unknown, if the value is not a valid integer, or if it is outside the Min/Max bounds listed above.)

---

## Points of Attention for the User

- A `VOLTAGE`/`MV` value of **0** indicates a **voltage measurement failure**, not an actual zero voltage (physically impossible during normal operation).
- The total safety shutdown (high or low thermal emergency) takes **absolute priority over all VDD/charging logic**: it applies even when voltage conditions would normally call for re-enabling the battery.
- The `TEMP_OFFSET_C` offset defaults to **-2 °C** (drift compensation); any value read via `GET TEMP` already includes this offset.
- Any change made via `SET` is **saved immediately and automatically** to Flash memory — it is therefore retained after a power loss.

---
