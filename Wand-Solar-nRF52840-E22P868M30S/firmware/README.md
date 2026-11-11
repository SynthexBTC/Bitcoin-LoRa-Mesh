# MeshCore ProMicro nRF52840 Custom Firmware

Based on MeshCore **v1.18.0** — the `ver` command shows **`v1.18.0-Synthex`**.

## Which Firmware to Install

Two versions are available. They have exactly the same features; only the clock differs.

| Version | USB install | Wireless update (OTA) | For which board |
|---|---|---|---|
| **Standard** | `ProMicro_repeater-v1.18.0-Synthex.uf2` | `ProMicro_repeater-v1.18.0-Synthex.zip` | Any ProMicro nRF52840. Use it if you are not sure. |
| **LFXO** | `ProMicro_repeater-v1.18.0-Synthex-LFXO.uf2` | `ProMicro_repeater-v1.18.0-Synthex-LFXO.zip` | Boards with a 32 kHz crystal, such as the **Wand PCB**. |

- **LFXO version:** keeps more accurate time. Recommended when the board has the crystal.
- **Standard version:** works everywhere, but its clock can drift slightly. The GPS corrects the time every 30 minutes.

> ⚠️ Do not install the LFXO version on a board without a 32 kHz crystal.

### Install over USB (`.uf2` file)
1. Connect the board over USB and press reset twice quickly: a USB drive appears.
2. Copy the `.uf2` file onto this drive. The board restarts by itself.

### Wireless update over Bluetooth (`.zip` file)
1. Copy the `.zip` file to your phone.
2. In the MeshCore app, log in to the repeater as admin, open the **Command Line** tab and type `start ota`. The repeater answers `OK`.
3. Open the **DFU** app on your phone, select the `.zip` file, then select the repeater and tap **Upload**.
4. Wait for the update to finish (a few minutes). Your settings are kept.

If the update fails, see the OTA section of the [MeshCore FAQ](https://github.com/meshcore-dev/MeshCore/blob/main/docs/faq.md).

To go back to the official firmware, install an official MeshCore `.uf2` or `.zip` file the same way.

## New Features

### GPS Auto-Cycle
GPS powers on automatically every **30 minutes**, gets a position, **sets the clock**, then powers off to save battery.
If no position is found within **2 minutes**, GPS powers off anyway and tries again at the next cycle.

| Situation | GPS | Comment |
|---|---|---|
| Between cycles | Off | About 30 min |
| Searching for position | On | Until a position is found, 2 min max |
| After `gps on` | Always on | The clock is still set automatically |
| After `gps off` | Off | GPS powers on again at the next cycle |

### Reliable GPS Detection at Startup
At startup, the repeater waits up to **5 seconds** for the GPS module, so slow modules (e.g. ATGM336H) are detected correctly.

### Works with Power Saving
The GPS auto-cycle keeps working when power saving is enabled, even without LoRa traffic.

## New Commands

### `gps pos`
Shows the last known GPS position, even when the GPS is off.

```
gps pos
→ lat:50.123456 lon:3.123456 alt:125m sats:8
  2026-06-11 14:32:07 UTC (last known)
```

| Field | Meaning |
|---|---|
| `lat` / `lon` | Latitude / longitude |
| `alt` | Altitude in meters |
| `sats` | Number of satellites |
| `UTC` | Date and time of the position |
| `(last known)` | GPS is off: last position found |
| `(saved prefs, ...)` | No GPS position since startup: saved position is shown |
| `no position available` | No GPS position and no saved position |

The last position is forgotten when the repeater restarts.

---

### `env`
Shows battery and sensor readings.

```
env
→ batt:70%/3.84v  temp:22.3C  hum:58%
```

| Field | Meaning |
|---|---|
| `batt` | Battery level (approximate) and voltage |
| `temp` | Temperature in °C (AHT10/AHT20 sensor) |
| `hum` | Humidity in % (AHT10/AHT20 sensor) |

If a sensor is not connected, its value is not shown.

---
