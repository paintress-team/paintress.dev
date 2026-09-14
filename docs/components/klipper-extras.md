# Klipper extras

`paintress-klipper-extras` adds inkjet printing to a
[Klipper](https://www.klipper3d.org/) machine. It loads jobs into the daemon,
calculates print bounds and coordinates gantry motion for each swath.

The plugin communicates with the daemon over TCP through `PaintressdClient`.
The daemon handles data streaming and communication with the firmware.

## `extras/paintress.py`

Registers the `PAINTRESS_*` G-code commands (connect, open payload, compute
print bounds, purge, trigger test, print and the rest), drives gantry motion
and runs the swath-by-swath print loop. A `[paintress]` config section
enables it.

For each swath it moves the head to the swath start, arms the swath, and
sweeps X at `print_speed` while the firmware fires. It does not stream swath
data; the daemon does that.

After each sweep, the plugin waits up to about 3 s for that swath's
`print_complete` event. This catches a missing trigger on the affected swath.
It polls the event stream in short intervals so Klipper's reactor stays
responsive.

## `extras/paintress_head.py`

This module registers optional `[paintress_head <name>]` sections. Select one
with `head:` in `[paintress]`. Each section stores the mounted head's offsets
and available purge channels. You can configure several heads and switch by
changing that selection.

The job's pass schedule defines the swath positions; the head section supplies
mounting offsets. The daemon separately checks that the job matches the
configured head.

## `extras/paintressd_client.py`

This synchronous TCP client lets `paintress.py` load jobs and arm swaths. It
can also run on its own to test daemon connectivity. A print still requires
Klipper motion and the hardware trigger.

## Print bounds and `exclude_object`

The plugin can use Klipper's `exclude_object` polygons to position an image
over a 3D-printed object. It derives the travel rectangle from the job's pass
schedule. Heads with stacked inks need lead-in passes below the image origin,
so travel can extend beyond the image bounds. See [Swaths and
passes](../concepts/swaths-and-passes.md).

## Failure model

A failure cancels the 2D job. The plugin aborts the firmware, returns the head
to its stored position and handles the error so an enclosing 3D print can
continue. The 2D job must restart from its first swath; it cannot resume
partway through.

## Start trigger

The plugin splits each sweep at `trigger_distance` and raises the
`[output_pin]` named by `trigger_output_pin` with `SET_PIN`. Both are set in
the `[paintress]` section. The motion MCU emits the edge at the scheduled
X position, avoiding TCP and USB timing jitter. Both moves follow the same
line, so the head passes through the split without stopping.