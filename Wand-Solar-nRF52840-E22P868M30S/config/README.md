# MeshCore Repeater Admin Configuration

## Initial Setup Commands

Run these commands via the CLI after flashing the firmware.

---

### Clock Synchronization
Sync the node clock from a connected companion device.
```
gps sync
```

---

### Power Management
Enable powersaving mode to reduce current consumption between LoRa events.
```
powersaving on
```

---

### Radio Configuration
Set frequency, bandwidth, spreading factor and coding rate.
```
set radio 869.618,62.5,8,8
```

| Parameter | Value | Description |
|---|---|---|
| Frequency | `869.618` MHz | EU868 band |
| Bandwidth | `62.5` kHz | |
| Spreading Factor | `8` | SF8 |
| Coding Rate | `8` | CR 4/8 |

---

### TX Power
Set transmit power in dBm.
```
set tx 22
```
> Maximum allowed value for the SX1262 in EU868: **22 dBm**.

---

### Duty Cycle
Set the airtime duty cycle limit.
```
set dutycycle 10
```
> 10% duty cycle — compliant with EU868 regulatory requirements.

---

## Full Setup Sequence

```
gps sync
set radio 869.618,62.5,8,8
set tx 22
set dutycycle 10
powersaving on
```
