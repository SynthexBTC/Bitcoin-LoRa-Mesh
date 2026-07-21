# Wand WatchDog

Battery **voltage** and **temperature** watchdog based on a PIC16F13113 microcontroller.

## What it does

This little module keeps an eye on your power and temperature, and takes care of your battery automatically.  
No buttons, no screen — it just runs quietly in the background.

Every 32 seconds, it wakes up, takes a quick look at the power level and the temperature, then goes back to sleep.

### When power is low

If the power stays low **3.4v** for a full 10 minutes, the battery turns off. It waits those 10 minutes on purpose, so a short, harmless dip in power doesn't trigger anything.
The battery remains connected to the charger so it can be recharged by the sun, with no further discharging.

### When power comes back

As soon as the power is good again **3.8v**, the battery turns back on right away.  
The system also gets a nRF52840 reset, just to make sure everything comes back up cleanly.

### When it's too cold or too hot

If the temperature drops below **-2°C** or climbs above **60°C**, charging is switched off to protect the battery. As soon as the temperature comes back into a normal range, charging switches back on. This check happens every single time it wakes up, so it reacts quickly in both directions.

## Power consumption

About half a microamp on average, low enough to run for years on a battery.

## Hardware

- Microcontroller: PIC16F13113 (also works on PIC16F13114 and PIC16F13115)
- Built with MPLAB X and the XC8 compiler
- Programmed with MPLAB SNAP (PG164100) on ICSP port

## Good to know

The voltage and temperature levels mentioned above are configurable in the firmware.  
They can be adjusted to fit your setup, but that requires reprogramming the module (a firmware recompile and reflash).
