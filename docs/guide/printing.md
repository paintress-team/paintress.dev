# Printing

Follow these steps to prepare a job, position it and print it through Klipper.

## 1. Prepare the job (offline)

```sh
# RIP: image -> halftoned passes (rip.json + rip.bin)
#   --head must match the head fitted to the machine, and the DPI must be
#   a multiple of that head's nozzle pitch (rip.py --list-dpi --head NAME).
python rip/rip.py photo.png -o rip.json --head c6n90 --dpi 630 --width 50

# Encode: RIP payload -> printer job (job.json + job.bin)
#   The encoder reads the head from the payload; it needs no --head.
python encoder/encoder.py rip.json -o job.json

# Optional: preview what will be printed, plus an ink estimate
python rip/viewer.py rip.json -o channels/ --soft-proof proof.png
```

Copy both `job.json` and `job.bin` into the plugin's `base_path` on the
printer host.

## 2. Connect

```gcode
PAINTRESS_CONNECT_SOCKET        ; TCP to the daemon
PAINTRESS_CONNECT_SERIAL        ; daemon opens the board's serial port
```

When the serial link opens, the daemon sends `IDENTIFY`. It rejects firmware
with an incompatible wire-protocol major version.

## 3. Load the payload

```gcode
PAINTRESS_OPEN_PAYLOAD FILENAME=job.json
```

The daemon validates the job and starts filling both firmware slots.
`profile_mismatch` means the job's frame differs from the firmware; reflash or
re-encode with matching definitions. `head_mismatch` means the job differs
from the daemon's `--head` setting; re-RIP for the fitted head or correct the
daemon setting.

The plugin calculates the firing interval from payload DPI and `print_speed`.
It sends that interval with each arm command, so a firmware reset cannot lose
it.

## 4. Print

There are two ways to position a print.

By default, the plugin uses Klipper's `exclude_object` polygons to position
the image over an object printed in the same session:

```gcode
PAINTRESS_PRINT
```

For an explicit origin, give the machine coordinates of the image's
bottom-left corner. Printing starts at the bottom of the image and progresses
in +Y:

```gcode
PAINTRESS_PRINT ORIGIN_X=20 ORIGIN_Y=30
```

The plugin checks the full travel rectangle against the axis limits, runs each
swath and returns the toolhead to its starting position. Progress and errors
appear in the console. On failure, it aborts the firmware before returning the
head.

## 5. Finish

```gcode
PAINTRESS_CLOSE_PAYLOAD
PAINTRESS_DISCONNECT_SERIAL
PAINTRESS_DISCONNECT_SOCKET
```

## Tips

Set `trigger_distance` to at least the acceleration distance, `v²/2a`. The
plugin warns if it is shorter. Starting to fire during acceleration compresses
the first columns.

Allow enough `end_overscan` for deceleration: 4 mm at the default 200 mm/s and
5000 mm/s².

Heads with stacked inks need lead-in travel below the image origin. If the
lower Y bound fails preflight, raise the print origin. `extra_margin` changes
image placement and should not be used for this clearance.

Prime the nozzles before you start: `PAINTRESS_PURGE CHANNEL=all PULSES=20`.

After successful abort/reset recovery, run `PAINTRESS_PRINT` again to restart
from the first swath. The job does not need to be reloaded. If recovery fails
or the cause is unclear, see [Troubleshooting](troubleshooting.md).