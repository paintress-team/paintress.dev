# System architecture

Paintress has five software modules in one repository. Four handle image
processing, motion, data transfer and firing. The fifth, `paintress-protocol`,
defines the shared wire protocol and head profiles and generates their
bindings. These interfaces must stay compatible across all modules. The
controller PCB is a separate KiCad project.

```
                  Image (PNG / JPG / ...)
                          │
                          ▼
           ┌──────────────────────────────┐
           │     paintress-rip-encoder    │   rip.py     → RIP payload (.json + .bin)
           │   raster image processing    │   encoder.py → job (.json + .bin)
           └──────────────────────────────┘
                          │  job (.json + .bin)
                          ▼
           ┌──────────────────────────────┐        motion / G-code
           │        Klipper host          │◄──── paintress-klipper-extras
           │  (3D-printer style gantry)   │   paintress.py + paintressd_client.py
           └──────────────────────────────┘
                          │  TCP, NDJSON protocol (default port 9000)
                          ▼
           ┌──────────────────────────────┐
           │       paintress-daemon       │   loads the job; double-buffer pipeline
           └──────────────────────────────┘
                          │  USB serial, framed binary protocol (CRC-16)
                          ▼
           ┌──────────────────────────────┐
           │      paintress-firmware      │   runs on the controller board MCU
           │  on the controller board PCB │   drives the piezo printhead
           └──────────────────────────────┘
                          │  analog drive: DAC waveforms, data bus, window selector
                          ▼
                   Piezo printhead
```

A Cartesian gantry driven by [Klipper](https://www.klipper3d.org/) moves the
printhead. The controller starts firing at the hardware trigger, then fires at
fixed intervals while the gantry holds a constant speed.

---

## The print pipeline

Each stage produces the input for the next.

1. **RIP.** `rip.py` loads an image, converts it to CMYK, applies dot gain
   compensation and halftones each channel into a 1-bit bitmap. The bitmap
   is sliced into passes, one printhead sweep each, and written out as a
   RIP payload: a `.json` header, a packed `.bin` file, and the Y
   position of every pass.

2. **Encode.** `encoder.py` reads that payload, applies the physical
   offsets between the head's nozzle columns, packs the head's buses into
   the 147-byte line the firmware expects, and writes the job as another
   `.json` header plus packed `.bin` file. The payload states which head
   it was made for, so the encoder packs for that head rather than for one
   compiled in at build time.

3. **Load.** The Klipper plugin tells the daemon to load the job. The daemon
   reads it into memory and starts pre-streaming swaths into the firmware's
   two PSRAM slots.

4. **Print.** For each swath the plugin moves the gantry to the swath start,
   arms the swath (the daemon holds the arm until the swath is in a slot),
   and sweeps X across the print area, raising the start trigger on the way.
   The firmware fires column by column as the head travels while the
   daemon's pipeline streams the swath after it.

5. **Fire.** The firmware schedules columns `line_delay_us` apart. It
   schedules the next firing time before doing per-line bookkeeping, keeping
   that work from shifting the grid. Even spacing on the substrate depends on
   constant gantry speed.

A **swath**, also called a pass, is the unit of work everywhere in the
stack: one horizontal strip printed in a single sweep of the head.

---

## Placement accuracy

The hardware trigger determines where the first column lands. The motion MCU
raises a GPIO line at a known X position, and the firmware waits for that edge
before firing. TCP and USB command delays therefore do not shift the start of
each pass.

After the trigger, firing is time-based. Accurate placement depends on a
constant, known gantry speed. The firmware does not correct speed variations
from belt resonance or microstepping. Address these on the motion side through
`[input_shaper]`, belt tension, sufficient overscan and microstep
interpolation.

---

## Keeping the pipeline in sync

The serial protocol, `ARM` parameters, daemon TCP API, head profiles and job
metadata must stay compatible across modules. `paintress-protocol` generates
the wire codes, payload structs and head scalars copied into the firmware,
daemon and encoder. Run `make sync-check` to check those copies. See [Serial
protocol](reference/serial-protocol.md) and [Changing the
protocol](dev/changing-the-protocol.md).