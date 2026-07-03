# WandWatchDog

Battery voltage watchdog based on a PIC16F13113 microcontroller.

## What it does

The chip watches the battery voltage and controls two signals: one that holds the
main CPU in reset, and one that enables the radio module.

At power-up, it releases the CPU, waits 3 seconds, then enables the radio.

After that, it spends almost all of its time asleep to save power, waking up
briefly every 32 seconds to check the battery:

- If the battery stays **below 3.4 V for 10 minutes**, it shuts everything down:
  the CPU is held in reset and the radio is disabled.
- If the battery stays **above 3.7 V for 2 minutes**, it starts everything again:
  the CPU is released first, and the radio is enabled 3 seconds later.
- Between 3.4 V and 3.6 V, nothing changes — this gap prevents the system from
  switching on and off repeatedly around a single threshold.

Short voltage dips or spikes are ignored: a change is only acted upon if it lasts
the full confirmation time.

## Power consumption

About half a microamp on average — low enough to run for years on a battery.

## Hardware

- Microcontroller: PIC16F13113 (also works on PIC16F13114 and PIC16F13115)
- Built with MPLAB X and the XC8 compiler
- Programmed with ICSP port
