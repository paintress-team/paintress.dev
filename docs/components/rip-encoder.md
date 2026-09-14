# RIP & Encoder

`paintress-rip-encoder` provides the Python tools that turn an image into a
print job. The RIP prepares the dot patterns; the encoder packs them for the
firmware.

## `rip/rip.py`, the raster image processor

Loads an image, converts it to CMYK (or CMYK plus light cyan and light
magenta), applies dot-gain compensation, and halftones each channel with one
of three dithering algorithms:

- Floyd–Steinberg error diffusion, the default,
- blue noise, higher quality, optional, needs `scipy`,
- ordered, Bayer or clustered-dot.

`--head` selects `c6n90` or `c4n180`, which defines the channels, nozzle pitch
and band step. Use `--ink-map` to specify the ink connected to each slot.

The output is a RIP payload, a `.json` header with a packed `.bin` file,
holding every halftoned pass and a metadata block: DPI, dimensions, channel
order, per-pass Y positions, and the head layout the payload was made for.
DPI can be any multiple of the head's nozzle pitch, 90 npi on `c6n90` and
180 on `c4n180`; `--list-dpi` prints the valid values.

Use `rip.py --target` to generate a calibration target without an input image:
`wedge`, `nozzle_check` or `col_align`. See [RIP and
halftoning](../concepts/rip-and-halftoning.md).

## `rip/viewer.py`, inspection

The viewer renders a preview of the RIP payload before encoding. Its soft
proof simulates drop spread to help inspect tone, grain, missing-nozzle
streaks and band seams. It also estimates ink use in µL per channel.

## `encoder/encoder.py`

The encoder reads the head layout from the RIP payload and converts the data
to the firmware wire format. It compensates for physical offsets between
nozzle columns, packs the buses into 147-byte lines and writes a `.json` job
header with a packed `.bin` file.

The job header carries two compatibility checks. The daemon compares
`geometry_fingerprint` with the firmware's frame fingerprint, and checks
`head_name` and `head_fingerprint` against its configured head. See [File
formats](../reference/file-formats.md) and [Printhead data
protocol](../concepts/printhead-protocol.md).

## `tools/`, calibration and bench

`calibrate_from_scan.py`, `nozzle_check_from_scan.py` and
`col_align_from_scan.py` turn a printed target into calibration data.

## Dependencies

`pillow` and `numpy`, plus optionally `numba` (JIT) and `scipy` (blue
noise).

## DPI, nozzles and passes

```
   passes_per_band = dpi / nozzle_pitch_npi
   lines_per_band  = band_step_nozzles * passes_per_band
```

A band is built by combining `passes_per_band` interleaved swaths.
`band_step_nozzles` is the shortest plumbed slot, not the column length. See
[Swaths and passes](../concepts/swaths-and-passes.md).
