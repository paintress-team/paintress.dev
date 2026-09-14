# Getting started

!!! warning "Experimental"
    Paintress has not been fully validated end to end on hardware. What
    follows is a development workflow, not a finished product. Read
    [Safety](safety.md) first.

This guide takes you from an image file to a print. Before starting, build the
firmware with the Raspberry Pi Pico SDK and flash it to the controller board.

## 1. RIP an image into halftoned passes

Writes `rip.json` + `rip.bin`.

```sh
python paintress-rip-encoder/rip/rip.py image.png -o rip.json \
    --head c6n90 --dpi 630
```

Set `--head` to the printhead fitted to your machine. DPI must be a multiple
of its nozzle pitch. Use `--list-dpi --head <name>` to list valid values.

## 2. Encode into a printer-ready job

Writes `job.json` + `job.bin`.

```sh
python paintress-rip-encoder/encoder/encoder.py rip.json -o job.json
```

## 3. Inspect what came out (optional)

Renders the RIP payload back to a preview image.

```sh
python paintress-rip-encoder/rip/viewer.py rip.json
```

## 4. Start the daemon

Run the daemon on the host connected to the controller board. It listens on
TCP port `9000` by default and opens the serial port on request.

```sh
cd paintress-daemon && python -m paintress_daemon --head c6n90
```

Set `--head` to the fitted printhead so the daemon can reject jobs made for
another head. The firmware cannot detect the physical head: both supported
profiles use the same firmware binary.

## 5. Run the print from Klipper

With `paintress-klipper-extras` installed and a `[paintress]` section
configured:

```gcode
PAINTRESS_CONNECT_SOCKET
PAINTRESS_CONNECT_SERIAL

PAINTRESS_OPEN_PAYLOAD FILENAME=job.json
PAINTRESS_PRINT                                ; uses exclude_object bounds
; or:
; PAINTRESS_PRINT ORIGIN_X=20 ORIGIN_Y=30      ; explicit origin
PAINTRESS_CLOSE_PAYLOAD

PAINTRESS_DISCONNECT_SERIAL
PAINTRESS_DISCONNECT_SOCKET
```

Without an explicit origin, the plugin uses Klipper's `exclude_object`
polygons to position the image over a 3D-printed object.

---

## Next steps

- [Installation](guide/installation.md) sets each piece up from scratch.
- [Anatomy of a print](concepts/anatomy-of-a-print.md) explains what those
  commands actually set in motion.
- [Troubleshooting](guide/troubleshooting.md) covers the usual failures.
