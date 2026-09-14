# Design rationale

This page explains the main design decisions and the constraints behind them.
Use it as context when proposing changes.

## Single firing method: the hardware start trigger

Early versions could start a swath after a timed delay or a hardware edge.
Host and USB latency made timed starts vary between passes, visibly shifting
the print. The motion MCU's hardware trigger starts at a known X position,
removing that source of error. Timed start and its settings were removed,
leaving one supported method.

## Time-based, open-loop firing

Columns fire at fixed time intervals while the gantry holds its cruise speed.
Position-based firing from a linear encoder or X step pulses would require
more wiring and firmware. For the intended use, the remaining speed variation
is expected to be small and repeatable, and can be addressed through input
shaping, belt tension and overscan. Revisit this decision if measurements show
that variation limits print quality.

## The daemon owns the streaming pipeline

Streaming a multi-megabyte swath inside a G-code handler would block Klipper's
reactor, starving motion planning and causing "Timer too close" shutdowns. The
daemon therefore streams ahead into two firmware slots. The plugin coordinates
movement, arming, sweeping and completion checks.

## One binary, several heads, and two fingerprints

The first design used one firmware build per head. Supporting a second head
showed that head identity and frame compatibility needed separate checks.

Originally, a single geometry fingerprint combined line size, nozzle count,
channel order and clock layout. It tied the encoder's packed output to a
firmware build.

The firmware does not use nozzle count or channel names to send the packed
data. Including those fields in its compatibility check made heads with the
same frame require separate builds.

The frame fingerprint now covers line size, clocks, edges and packing and is
exchanged at `IDENTIFY`. Both supported heads share it and use one firmware
binary. The head fingerprint covers name, nozzle count and channels; the
daemon checks it against `--head`.

Per-bus nozzle mappings and PIO programs remain code; profile scalars are
generated. The generator processes all profiles together and rejects a head
with an incompatible frame rather than producing a partially compatible
firmware header.

This depends on correct machine configuration. The board cannot identify the
physical head, so the daemon's `--head` must match it. A job for another head
can pass the frame check when both share a frame.

## Pack for the wire, expand for DMA

Each line uses 147 bytes over USB and expands to 196 bytes for PIO/DMA.
Packing reduces transfer size by 25%; expanding on receipt removes repacking
from the firing loop. The encoder and firmware must agree on both layouts.

## Fail loudly, fail early

Compatibility checks reject unsupported sessions or jobs before printing.
Failed transfers are discarded, and print faults stay latched until recovery.
`ABORT` latches the DAC off; further arm requests return `DAC_LATCHED` until
reset. These checks keep invalid or incomplete data from being reported as a
successful print.