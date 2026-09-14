# Firing and timing

Paintress places drops by timing nozzle firings while the gantry moves at
constant speed. A hardware trigger sets the first column's position; a fixed
time interval sets the spacing of the remaining columns.

## The fixed firing grid

The column interval follows from the payload DPI and the print speed:

```
line_delay_us = 25 400 000 / (dpi × print_speed_mm_s)
```

At 630 dpi, the interval is 202 µs at the default 200 mm/s, or 403 µs at 100
mm/s. Every `ARM` carries the interval, so a device reset does not lose a
separate timing setting.

The print loop schedules the next tick before doing per-line bookkeeping. This
prevents processing delays from accumulating between columns. After one column
fires, DMA transfers the next column's data within the firing interval.

## The hardware start trigger

The time grid is measured from the first column, so that column must start at
a known X position. A start command sent over TCP and USB would introduce
timing jitter. Paintress uses this sequence:

1. `ARM` has the firmware do all of its setup and then wait for a rising
   edge on its trigger input.
2. The plugin splits the sweep at `trigger_distance` and raises a Klipper
   `[output_pin]` at the split, so the edge comes from the motion MCU at
   the precise print time X reaches that position. The two halves of the
   sweep are colinear, so the head cruises through without stopping.
3. The firmware samples the trigger line only inside the armed window, and
   establishes a known-low baseline first, so a stuck-high or noisy line
   cannot start a print. If the edge never arrives it gives up after a
   guard timeout and reports `TRIGGER_TIMEOUT` to the host.

The line is forced low before each pass and dropped again after every sweep,
so a pass always sees exactly one clean rising edge.

## Open-loop accuracy, and its limits

The trigger fixes the start position. After that, the firmware assumes the
gantry holds its commanded speed. It does not measure or correct velocity
ripple. Use input shaping, belt tension, sufficient overscan and microstep
interpolation to address motion errors.

Position-based firing using an encoder or X step pulses was considered. The
current design uses timed firing because the expected residual motion error is
small and repeatable for the intended use. See [Design
rationale](../dev/design-rationale.md).

## Speed budget

Each column needs about 12 µs for the reference waveform and 33 µs for the
next data-bus DMA transfer. The minimum interval is about 46 µs, equivalent to
roughly 875 mm/s at 630 dpi. The default 200 mm/s is well below this rate. The
plugin accepts derived intervals of 46–65535 µs; the firmware itself only
rejects zero.

The [double buffer](double-buffering.md) spreads USB transfers across a swath.
USB does not set the per-column firing interval, though the next swath must be
fully received before it can be armed.