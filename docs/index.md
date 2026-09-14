# Paintress

**Paintress** is an open-source controller stack for **piezoelectric inkjet
printheads**. It converts images into print data, coordinates printing with
Klipper, and drives the printhead through dedicated firmware and a controller
board.

!!! warning "Status — Experimental / under active development"
    Paintress is **not ready for use**. The code is not clean or organised
    and it changes without notice. Expect rough edges and breakage.

!!! danger "Safety"
    The hardware drives reactive loads (piezoelectric elements) at voltages
    up to **42 V DC**, and it has **not** been certified to any safety
    standard. Read the [Safety](safety.md) page before you build or power
    anything.

!!! note "Independent project"
    Paintress is not affiliated with, endorsed by or otherwise connected to
    the [Klipper](https://www.klipper3d.org/) project. It uses Klipper as its
    motion platform.

---

## What Paintress does

A Cartesian gantry running [Klipper](https://www.klipper3d.org/) moves the
printhead across the substrate. A hardware trigger starts each pass at a known
X position; the controller then fires the nozzles at fixed intervals while the
gantry holds its print speed.

<div class="grid cards" markdown>

-   :material-image-filter-center-focus: **RIP & Encoder**

    Turn an image into halftoned, printer-ready data.
    [:octicons-arrow-right-24: Details](components/rip-encoder.md)

-   :material-lan-connect: **Daemon**

    Stream jobs to the firmware with a double-buffer pipeline.
    [:octicons-arrow-right-24: Details](components/daemon.md)

-   :material-printer-3d: **Klipper extras**

    Move the printhead and coordinate each pass.
    [:octicons-arrow-right-24: Details](components/klipper-extras.md)

-   :material-chip: **Firmware**

    Dual-core RP2350 print engine firing on a fixed grid.
    [:octicons-arrow-right-24: Details](components/firmware.md)

</div>

---

## Start here

New to the project? Read the [Architecture](architecture.md) page, then
[Anatomy of a print](concepts/anatomy-of-a-print.md).

For a first print, follow [Getting started](getting-started.md),
[Installation](guide/installation.md) and [Printing](guide/printing.md). Use
[Troubleshooting](guide/troubleshooting.md) to diagnose problems.

For software integration, see the [Serial
protocol](reference/serial-protocol.md) and [Daemon TCP
API](reference/tcp-api.md).

---

## Support the project

Paintress is developed and tested independently. Donations help pay for ink,
printheads, boards and bench testing. You can support the work on Ko-fi:

[:simple-kofi: Donate on Ko-fi](https://ko-fi.com/paintressteam){ .md-button .md-button--primary }

You can also help by testing hardware, reporting issues, or contributing code
and documentation.