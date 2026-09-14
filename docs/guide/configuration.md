# Configuration

Configure a trigger `[output_pin]` and a `[paintress]` section in Klipper. You
can also add `[paintress_head <name>]` sections to store offsets and purge
channels for each mounted head.

## The trigger output pin

Declare the motion-MCU pin that is wired to the board's trigger input as an
ordinary Klipper output pin:

```ini
[output_pin paintress_trigger]
pin: PE6            # any free pin on the motion MCU
value: 0
```

The plugin uses `SET_PIN` to control this output. It sets the line low before
each pass and lowers it again after the sweep.

## The `[paintress]` section

```ini
[paintress]
serial_port: /dev/serial/by-id/usb-...-if00
base_path: /home/pi/printer_data/paintress/
x_offset: 41.0
y_offset: -30.0
z_offset: 1.0
print_speed: 200
trigger_output_pin: paintress_trigger
#head: c4n180
#trigger_distance: 10.0
#verbose: false
#auto_connect: true
```

!!! warning "Which way the offsets point"
    `x_offset` and `y_offset` point from the printhead reference to the
    toolhead reference. If the head sits 30 mm in +X from the toolhead
    nozzle, use `x_offset: -30`. Reversing the sign shifts the image by
    twice the offset. To calibrate, print a target, measure its signed
    displacement from the intended position, and subtract that displacement
    from the current offset.

| Option | Default | Meaning |
|---|---|---|
| `socket_port` | `9000` | TCP port of the daemon, assumed to be on localhost. |
| `serial_port` | — | Serial device the daemon opens to reach the board. Use a stable `/dev/serial/by-id/...` path. |
| `base_path` | `/home/pi/printer_data/paintress/` | Root directory for payloads. `PAINTRESS_SET_DIRECTORY` is sandboxed under it. |
| `auto_connect` | `true` | Open the daemon socket and the board serial link once Klipper is ready. Best-effort: a daemon that is not up yet gets a warning, not an aborted startup. |
| `x_offset`, `y_offset` | `0.0` | Distance **from** the printhead's reference point **to** the toolhead reference point, in mm. See the warning above. |
| `z_offset` | `1.0` | Raise applied to bring the head to working height, and also the travel clearance (mm). |
| `z_lift_speed` | `5.0` | Z move speed (mm/s). |
| `move_speed` / `move_accel` | `300` / `5000` | Travel (non-printing) motion limits. |
| `print_speed` / `print_accel` | `200` / `5000` | Printing-pass motion limits. `print_speed` also sets the firing interval. |
| `start_overscan` | `10.0` | Acceleration distance before the print area (mm). |
| `end_overscan` | `10.0` | Distance after the print area (mm). Must cover deceleration at `print_speed`. |
| `extra_margin` | `2` | Bleed added around `exclude_object` bounds when printing over an object (mm). It moves the origin, so never raise it to buy travel clearance for a head's lead-in. |
| `head` | *(none)* | Names a `[paintress_head <name>]` section. Optional; without it, the plugin uses `[paintress]` offsets and accepts every channel name. |
| `trigger_output_pin` | *(required)* | Name of the `[output_pin]` wired to the board's trigger input. |
| `trigger_distance` | `start_overscan` | X travelled from the pass start before the rising edge fires (mm). Has to be at least the acceleration distance and less than the swath length. |
| `verbose` | `false` | Echo per-move `OK G1 ...` lines to the console. |

## Head sections (optional)

Set `head:` to a `[paintress_head <name>]` section to use its mounting offsets
and available purge channels. With several sections configured, changing this
one setting selects the offsets for a different head.

```ini
[paintress]
head: c4n180

[paintress_head c4n180]
# Offsets are ILLUSTRATIVE -- every carriage differs. Calibrate them.
x_offset: 1.5
y_offset: -2.25
z_offset: 3.0
channels: all, yellow, black, magenta, cyan

[paintress_head c6n90]
channels: all, yellow, black, light_cyan, light_magenta, magenta, cyan
```

You can declare heads that are not currently fitted. Omitted values keep their
existing settings. Install `paintress_head.py` alongside `paintress.py`;
without it, Klipper reports `Section 'paintress_head c4n180' is not a valid
config section`.

`channels` lists the purge channels available on the head. Their firmware
indices are fixed. The plugin rejects a purge request for an unavailable
channel instead of running a purge that fires no nozzles.

The head section supplies mounting offsets; the job's pass schedule defines
the swath positions. The daemon separately rejects jobs made for a different
configured head.

!!! tip "Timing is derived, not configured"
    The firing interval is computed from the payload DPI and `print_speed`,
    `25 400 000 / (dpi × speed)`, and travels inside every arm. It has to
    fit in 1–65535 µs, so very low DPI × speed combinations are rejected
    with a clear error, and an interval below the firmware's per-column
    floor of about 46 µs gets a warning.

## Daemon

The daemon uses command-line options rather than a config file. Set the fitted
printhead when starting it:

```sh
python -m paintress_daemon --head c4n180
```

Use the head that is physically fitted. The firmware cannot identify it, so
the daemon relies on `--head` to reject incompatible jobs.

The daemon listens on `127.0.0.1:9000` and uses 2 000 000 baud by default. The
API has no authentication and can fire ink or reset hardware. Use `--host` to
expose it only on a trusted network. The `connect` command selects the serial
port, using the plugin's `serial_port` setting.