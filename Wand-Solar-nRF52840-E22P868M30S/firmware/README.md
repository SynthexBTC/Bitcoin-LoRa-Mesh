# MeshCore ProMicro nRF52840 Custom Firmware



## New Features

### GPS Auto-Cycle
GPS powers on automatically every **30 minutes**, acquires a fix, **syncs the system clock**, then powers off.  
Maximum time to obtain a fix: **2 minutes** (timeout — GPS powers off regardless).

| State | GPS | Comment |
|---|---|---|
| Between cycles | Off | Duration ~29 min |
| Acquiring fix | On | Duration ~15 sec (warm start) |
| Manual `gps on` | Always on | Clock synced but GPS never powered off by cycle |
| Manual `gps off` | Off | GPS Auto-Cycle is always on |

### Boot GPS Detection
At boot, firmware waits up to **5 seconds** for the GPS module to send its first NMEA sentence, instead of the original fixed 1-second delay. Prevents GPS from being marked as undetected on slow-starting modules (e.g. ATGM336H which needs ~2s).

### Powersaving Compatibility
A `SoftwareTimer` wakes the CPU every **60 seconds** to check the GPS cycle timer, ensuring the auto-cycle triggers even when no LoRa traffic is present.

### LFXO Clock Source
Switched from internal RC oscillator (LFRC) to external 32kHz crystal (LFXO) for accurate `millis()` timing during sleep — required for reliable GPS cycle timing.
> A good 32kHz crystal is present and functional on the Wand PCB.



## New CLI Commands

### `gps pos`
Displays the last known GPS position, even if the GPS is currently off.

```
gps pos
→ lat:50.123456 lon:3.123456 alt:125m sats:8
  2026-06-11 14:32:07 UTC (last known)
```

| Output field | Description |
|---|---|
| `lat` / `lon` | Coordinates in decimal degrees |
| `alt` | Altitude in meters |
| `sats` | Number of satellites at time of fix |
| `UTC` | Date and time of the fix (from NMEA, not system clock) |
| `(last known)` | GPS is currently off — position retained from last fix |
| `(saved prefs)` | No GPS fix ever — position from saved preferences |

---

### `env`
Displays battery and environmental sensor data.

```
env
→ batt:70%/3.84v  temp:22.3C  hum:58%
```

| Output field | Description |
|---|---|
| `batt` | Battery percentage (LiPo: 3000mV=0%, 4200mV=100%) and voltage |
| `temp` | Temperature in °C (1 decimal place) |
| `hum` | Relative humidity in % (no decimal) |

---
