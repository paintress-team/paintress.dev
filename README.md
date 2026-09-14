<p align="center">
  <img src="docs/assets/paintress.png" alt="Paintress" width="160">
</p>

# Paintress

Paintress is an open-source controller stack for piezoelectric inkjet
printheads. It converts images into print data, coordinates motion through
Klipper, and drives the head with RP2350 firmware and a dedicated controller
board.

This repository contains the documentation site. The five software modules
live together in the Paintress code repository; the controller PCB is a
separate KiCad project.

> **Experimental — not ready for use.** The code is under active development,
> needs cleanup and organisation, and may change without notice. Hardware
> validation is incomplete.

> **Safety.** The hardware drives reactive piezoelectric loads at up to
> **42 V DC** and has not been certified to any safety standard. It is
> provided for educational and research purposes. Do not leave powered
> hardware unattended, and treat the head as energised whenever the board
> is powered. Read [Safety](docs/safety.md) before building or operating it.

## Table of contents

- [System architecture](#system-architecture)
- [The print pipeline](#the-print-pipeline)
- [The modules](#the-modules)
- [Key concepts](#key-concepts)
- [File and protocol formats](#file-and-protocol-formats)
- [End-to-end example](#end-to-end-example)
- [Glossary](#glossary)
- [Project layout](#project-layout)
- [Support](#support)
- [License](#license)

## System architecture

Klipper moves the printhead across the substrate. The plugin coordinates
each pass with the daemon, which sends data over USB to the controller.
The firmware starts firing on a hardware trigger from the motion MCU, then
fires at fixed intervals while the gantry holds its print speed.

`paintress-protocol` defines the wire protocol and head profiles shared by
the other modules. It generates C and Python bindings; `make sync-check`
checks that the copies used by each module match those definitions.

See [System architecture](docs/architecture.md) for the data flow and
[Anatomy of a print](docs/concepts/anatomy-of-a-print.md) for the sequence of
actions during a job.

## The print pipeline

1. **RIP.** `rip.py` converts the image to ink channels, compensates for dot
   gain, halftones it and divides it into passes. It writes `rip.json` and
   `rip.bin`, including each pass's Y position.
2. **Encode.** `encoder.py` reads the head layout from the payload, compensates
   for nozzle-column offsets and packs the data into 147-byte lines. It writes
   `job.json` and `job.bin`.
3. **Load.** The plugin asks the daemon to load the job. The daemon checks
   compatibility and starts filling the firmware's two PSRAM slots.
4. **Print.** The plugin moves to each pass's start, arms it, sweeps X
   and raises the trigger at `trigger_distance`. It checks completion after
   the sweep while the daemon prepares more data.
5. **Fire.** The firmware schedules columns `line_delay_us` apart. Even dot
   spacing depends on the gantry maintaining constant speed.

A swath, or pass, is one strip printed in a single sweep. An interrupted 2D
job must restart from its first swath. The plugin handles the error so an
enclosing 3D print can continue.

## The modules

### `paintress-rip-encoder`

Python tools for preparing print jobs:

- `rip/rip.py` converts images to CMYK or CMYK with light cyan and light
  magenta. It supports Floyd–Steinberg dithering (default), blue noise and
  ordered dithering. `--head` selects the layout; `--ink-map` records the
  ink connected to each slot.
- `rip/viewer.py` previews RIP output, produces a soft proof and estimates
  ink use.
- `encoder/encoder.py` packs the payload for its head and writes the job files.
- `tools/` contains tone, nozzle and column-alignment calibration utilities.

Dependencies are `pillow` and `numpy`, with optional `numba` for JIT
acceleration and `scipy` for blue noise. See
[RIP & Encoder](docs/components/rip-encoder.md) and
[RIP and halftoning](docs/concepts/rip-and-halftoning.md).

### `paintress-daemon`

The daemon loads jobs, manages USB serial communication and keeps the
firmware supplied with swath data. It serves NDJSON over TCP on port `9000`
by default and forwards firmware events and logs to clients. Gantry motion
stays in Klipper.

An `asyncio` server handles clients. Commands that change state run in one
worker thread; read-only requests use a thread pool. Separate RX and TX
threads handle serial communication, accumulating encoded frames into 64 KB
writes. Urgent abort and reset frames bypass the normal TX queue.

Start it with the fitted head selected:

```sh
python -m paintress_daemon --head c6n90
```

Unexpected serial loss cancels the print. The daemon waits for explicit
`reconnect`, `reset` or `abort` recovery and does not retry on its own.
See [Daemon](docs/components/daemon.md).

### `paintress-klipper-extras`

The Klipper plugin has three files:

- `extras/paintress.py` registers the `PAINTRESS_*` commands, calculates bounds
  and runs the motion loop. Enable it with a `[paintress]` section.
- `extras/paintress_head.py` registers optional sections for mounting offsets
  and available purge channels.
- `extras/paintressd_client.py` connects to the daemon over TCP. It can also
  run on its own to test connectivity.

The plugin can use `exclude_object` polygons to position an image over a
3D-printed object, or accept an explicit origin. It reaches the firmware
through the daemon. See [Klipper extras](docs/components/klipper-extras.md)
and [Configuration](docs/guide/configuration.md).

### `paintress-firmware`

C firmware for a dual-core RP2350 with 8 MB of external PSRAM:

- Core 0 handles USB, frame decoding, swath storage and host events.
- Core 1 runs the print engine and purge operations.
- Two PSRAM slots allow receiving one swath while printing another.
- PIO and DMA drive the data bus, window selector and 10-bit DAC interface.

Each `ARM` supplies its firing interval. `ABORT` disables and latches off the
DAC while an active swath loop finishes without ink. `RESET` reboots the
chip and clears its state. A watchdog also reboots an unresponsive system;
an unexpected reboot cancels the print.

The integrated thermistor has not been characterised. Its conversion values
are placeholders, and the thermal monitor is observe-only by default.
**Assume there is no thermal protection.** See
[Firmware](docs/components/firmware.md).

### `paintress-protocol`

This module defines the shared interfaces:

- `schema/protocol.yaml`: wire codes and payload structures, with generated
  C and Python bindings.
- `profiles/*.yaml`: head geometry, frame settings and reference ink maps.
- `job/paintress_job.py`: the job container shared by the encoder and daemon.

Jobs carry two fingerprints. The **frame fingerprint** checks the packed
format against the firmware. The **head fingerprint** checks the job against
the daemon's configured head. Both supported heads share one frame and one
firmware binary; the board cannot detect which head is fitted.

Run `make sync-check` to verify generated copies. See
[Changing the protocol](docs/dev/changing-the-protocol.md).

### The controller board

The controller PCB and printhead adapter are maintained in a separate KiCad
project. The firmware's `config/board_config.h` defines the expected GPIO map.
Verify it against your board revision before wiring.

The board was developed through independent reverse engineering using
oscilloscope measurements, signal analysis and public references. No
proprietary documentation or firmware was used. See
[Controller board](docs/hardware/controller-board.md).

## Key concepts

| Concept | Meaning |
|---|---|
| Swath / pass | One horizontal strip printed in a single sweep. |
| Band | Consecutive image rows covered by interleaved passes. |
| Column | Nozzle firings at one X position. |
| Channel | One ink colour, defined by the head layout. |
| Head slot | One ink feed and its contiguous set of nozzles. |
| Bus | One data pin carrying 180 two-bit nozzle codes and a 32-bit window map. |
| DPI | Dots per inch; must be a multiple of nozzle pitch in npi. |
| PSRAM slot | One of two buffers, each holding a complete swath. |
| Overscan | Travel before and after the image for acceleration and deceleration. |

Passes are interleaved to reach the selected DPI:

```text
passes_per_band = dpi / nozzle_pitch_npi
lines_per_band  = band_step_nozzles * passes_per_band
```

`band_step_nozzles` is the shortest plumbed slot. On `c6n90`, 630 dpi uses
seven passes per band. Heads with stacked ink slots, such as `c4n180`, need
lead-in passes below the image. See
[Swaths and passes](docs/concepts/swaths-and-passes.md).

## File and protocol formats

| Format | Producer → consumer | Contents |
|---|---|---|
| `rip.json` + `rip.bin` | RIP → encoder | Metadata, head layout, pass positions and bit-packed halftone data. |
| `job.json` + `job.bin` | Encoder → daemon | Fingerprints, dimensions, pass index and packed 147-byte lines. |
| NDJSON over TCP | Plugin ↔ daemon | Requests, responses matched by `id`, and asynchronous messages. |
| Framed binary over USB CDC | Daemon ↔ firmware | Commands, data, ACK/NACK, events and logs, protected by CRC-16-CCITT. |

Both file pairs use a JSON header and a CRC-32-protected binary file. Each
147-byte job line expands to 196 bytes in PSRAM for DMA. A swath transfer
consists of `BEGIN_SWATH`, its `DATA` frames and `END_SWATH`.

See [File formats](docs/reference/file-formats.md),
[Daemon TCP API](docs/reference/tcp-api.md),
[Serial protocol](docs/reference/serial-protocol.md) and
[Printhead data protocol](docs/concepts/printhead-protocol.md) for field layouts
and command tables. Change shared definitions in `paintress-protocol` and
regenerate all consumers together.

## End-to-end example

First build and flash the firmware, install the daemon and plugin, and
configure `[paintress]` with its trigger output pin. See
[Installation](docs/guide/installation.md).

Prepare an image for a machine fitted with `c6n90`:

```sh
python paintress-rip-encoder/rip/rip.py image.png -o rip.json --head c6n90 --dpi 630
python paintress-rip-encoder/encoder/encoder.py rip.json -o job.json
python paintress-rip-encoder/rip/viewer.py rip.json
```

Use `--list-dpi --head <name>` to list valid resolutions for another head.
Copy both job files to the printer host's `base_path`, then start the daemon
on that host:

```sh
cd paintress-daemon
python -m paintress_daemon --head c6n90
```

From the Klipper console:

```gcode
PAINTRESS_CONNECT_SOCKET
PAINTRESS_CONNECT_SERIAL
PAINTRESS_OPEN_PAYLOAD FILENAME=job.json
PAINTRESS_PRINT ORIGIN_X=20 ORIGIN_Y=30
PAINTRESS_CLOSE_PAYLOAD
PAINTRESS_DISCONNECT_SERIAL
PAINTRESS_DISCONNECT_SOCKET
```

Omit both origin arguments to use `exclude_object` bounds. See
[Getting started](docs/getting-started.md) and [Printing](docs/guide/printing.md)
for the workflow, and [Troubleshooting](docs/guide/troubleshooting.md) for
recovery steps.

## Glossary

The [Glossary](docs/reference/glossary.md) defines the terms used throughout
the documentation, including head slots, PSRAM slots, firing windows and the
two compatibility fingerprints.

## Project layout

| Location | Contents |
|---|---|
| Paintress code repository | The five software modules, internal `docs/` and module reviews in `analises/`. Each module has its own README. |
| `paintress-team/paintress.dev` | This documentation site, with pages in `docs/`, published at [paintress.dev](https://paintress.dev). |
| Separate KiCad project | Controller and printhead adapter boards. |

For software tests and documentation builds, see
[Building and testing](docs/dev/building-and-testing.md).

## Support

Donations help fund ink, printheads, boards and bench testing. You can
[support Paintress on Ko-fi](https://ko-fi.com/paintressteam), report issues,
test hardware or contribute code and documentation.

## License

Licensing for the code modules is incomplete. `paintress-daemon/LICENSE` is
MIT, and `paintress-klipper-extras/extras/paintress_head.py` declares GPLv3.
Check each module's files before using, modifying or redistributing it.

The documentation has separate licenses: CC BY 4.0 for the text and MIT for
the site code. The published `paintress-team/paintress.dev` repository carries
both license files.
