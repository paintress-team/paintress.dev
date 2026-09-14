# Firmware

`paintress-firmware` runs on a dual-core RP2350 with 8 MB of external PSRAM
mapped through XIP. The default `PICO_BOARD` in `CMakeLists.txt` is
`solderparty_rp2350_stamp_xl`. It runs at an overclocked 250 MHz, with the
core voltage raised before the clock speed.

Heads that share the same wire frame use one firmware binary. The generated
header defines line size, clocks, edges and packing. Head name and nozzle
count are checked on the host, since the board cannot detect the fitted head.

## Dual-core split

The cores communicate through two SPSC queues: commands from core 0 to core 1,
and events in the reverse direction.

Core 0 owns the USB link. It receives commands and swath data from the
daemon, decodes frames with CRC-16-CCITT, drives the swath manager and
forwards core 1's events to the host.

Core 1 runs the print engine. It brings up the printhead drivers and then
loops on the inter-core command queue, executing arms (prints) and purges.

## Fixed firing grid

The print loop schedules one column every `line_delay_us`. Each `ARM` carries
this interval, so it does not need to be configured separately on the device.
The next column's data DMA starts after the current column fires. Scheduling
the next tick before per-line bookkeeping prevents that work from delaying the
grid.

## PSRAM double buffer

The 8 MB PSRAM is divided into two equal slots. One holds the printing swath
while the other receives the next. Each packed 147-byte USB line expands to
196 bytes in PSRAM: 49 32-bit words.

## Printhead drive

Three PIO and DMA interfaces drive the head:

- **DAC**: streams drive waveform samples (rise, default, fall) over a
  10-bit parallel bus, with two clock pins selectable at runtime.
- **Data bus**: a DDR-style parallel bus, packing four 6-bit values into
  each 32-bit word the DMA pushes to PIO.
- **Window selector**: a 3-bit selector, latch / A / B.

An ADC reads the head's integrated thermistor. Its curve is unknown, with no
datasheet and no reverse engineering done on it yet, so the conversion
parameters are placeholders. The thermal
monitor only logs readings by default; it does not enforce temperature limits.

## Starting a swath: hardware trigger

`ARM` latches the window selector, ramps up the DAC and preloads line 0. The
firmware then waits for a rising edge on `TRIGGER_IN_PIN`. It samples the pin
only while armed, after establishing a low baseline. If no edge arrives,
`DEFAULT_TRIGGER_TIMEOUT_US` limits the wait.

## Stopping, resetting and the watchdog

`ABORT` cuts the electrical drive while the current swath loop continues.
`RESET` and the watchdog reboot the chip.

- **`ABORT`** is a purely electrical kill-switch on core 0. It drops the DAC
  standby and reset pins and latches the DAC off across both cores. A swath
  already firing runs to its natural end dry, and `ARM` and `PURGE` are
  NACKed `DAC_LATCHED` until recovery.
- **`RESET`** is a full chip reboot. The firmware ACKs, drains the TX and
  reboots through the watchdog, clearing slots, the DAC latch and all other
  state. USB re-enumerates in a second or two; the daemon waits and reconnects.
- A **hardware watchdog**, fed by core 0 only while core 1's heartbeat
  advances, reboots the chip when either core wedges. That drop is
  unexpected, so it cancels the print.

A commanded reboot leaves a marker in watchdog scratch register 0. `IDENTIFY`
reports `boot_flags` so the host can distinguish a commanded reset (bit 0)
from watchdog fault recovery (bit 1).

Thermal enforcement remains disabled (`SAFETY_ENFORCE 0`) until the thermistor
has been characterised.

## Build and flash

With the Raspberry Pi Pico SDK in place (`PICO_SDK_PATH` set), run
`cmake .. && make -j` in `build`. The output is `paintress_firmware.uf2`,
flashed over BOOTSEL mass storage or with `picotool load -v -x`. The
firmware emits no `printf` over USB; host output arrives as `LOG` frames on
the same CDC the daemon uses. Pure modules are host-tested under `tests/`
with CTest.
