# G-code reference

Commands report `OK <command>` or `ERROR <command>: ...` in the console. A
Paintress command error does not abort the Klipper session.

## Connection

| Command | Arguments | Description |
|---|---|---|
| `PAINTRESS_CONNECT_SOCKET` | — | Open the TCP connection to the daemon. |
| `PAINTRESS_DISCONNECT_SOCKET` | — | Close the TCP connection. |
| `PAINTRESS_CONNECT_SERIAL` | — | Ask the daemon to open the board's serial port (`serial_port`). |
| `PAINTRESS_DISCONNECT_SERIAL` | — | Ask the daemon to close the serial port. |
| `PAINTRESS_STATUS` | — | Report combined daemon and plugin status as JSON. |

## Payload

| Command | Arguments | Description |
|---|---|---|
| `PAINTRESS_SET_DIRECTORY` | `DIRECTORY=` | Select the payload search directory, sandboxed under `base_path`. |
| `PAINTRESS_OPEN_PAYLOAD` | `FILENAME=` | Load a job into the daemon and configure the firing interval. |
| `PAINTRESS_CLOSE_PAYLOAD` | — | Unload the current job. |

## Printing

| Command | Arguments | Description |
|---|---|---|
| `PAINTRESS_GET_BOUNDS` | `FROM_MESH=` (default `1`) | Compute the print bounds from `exclude_object` (`1`) or from the payload size (`0`). |
| `PAINTRESS_PRINT` | `ORIGIN_X=` `ORIGIN_Y=` (optional, together) | Print all swaths in the job. With no origin, bounds come from `exclude_object`. |

## Maintenance and testing

| Command | Arguments | Description |
|---|---|---|
| `PAINTRESS_PURGE` | `CHANNEL=` (default `all`) `PULSES=` (default `10`) | Fire a channel's nozzles to clear or prime ink. |
| `PAINTRESS_TEST_TRIGGER` | `PIN=` `MOVE=` `DISTANCE=` `TRIGGER_DISTANCE=` `SPEED=` `ACCEL=` `PULSE_MS=` (all optional) | Pulse the trigger `[output_pin]` for oscilloscope checks, without using the daemon or firmware. `MOVE=1` fires the edge at a colinear split in an X move, mirroring a real pass; `MOVE=0` pulses in place. |
| `PAINTRESS_SET_TRIGGER_DISTANCE` | `VALUE=` (mm, optional) | Read (no `VALUE`) or set the trigger distance at runtime. Lasts until restart. |

## The `CHANNEL` argument

`CHANNEL` accepts a name or number from the table below. Indices match the
firmware's `channel_masks[]` table and stay the same across heads. A
`[paintress_head]` section's `channels:` list restricts purging to channels
available on that head.

| Name | Index |
|---|---|
| `all` | `0` |
| `yellow` | `1` |
| `black` | `2` |
| `light_cyan` | `3` |
| `light_magenta` | `4` |
| `magenta` | `5` |
| `cyan` | `6` |
