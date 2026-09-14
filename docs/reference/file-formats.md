# File formats

The pipeline writes two file pairs: a RIP payload and an encoded print job.
Each has a versioned JSON header and a packed binary file protected by CRC-32.

## RIP payload: `rip.json` + `rip.bin`

`rip.py` writes this pair and `encoder.py` reads it. The binary contains each
halftoned pass as 1-bit bitmaps packed with `np.packbits`.

```jsonc
// rip.json  (sibling: rip.bin)   -- the example is c6n90
{
  "format": "paintress-rip",
  "format_version": "0.2.0",
  "metadata": {
    "dpi": 630, "image_width_px": 0, "nozzle_count": 90,
    "passes_per_band": 7,
    // which head this payload was made for; the encoder packs for it
    "head_layout": {
      "name": "c6n90", "nozzle_pitch_npi": 90, "column_length": 90,
      "channel_order": ["C", "M", "Y", "K", "LC", "LM"],
      "ink_map": ["K", "Y", "LM", "LC", "C", "M"],
      "band_step_nozzles": 90, "lead_in_bands": 0, "lead_in_mm": 0.0,
      "slots": [ { "column": 0, "first_nozzle": 0, "nozzle_count": 90,
                   "physical_nozzle_count": 90, "ink": "K" } ]
    },
    "ink_map": ["K", "Y", "LM", "LC", "C", "M"],
    "processing": {}
  },
  "passes": { "y_positions_mm": [], "y_deltas_mm": [] },
  "array": { "shape": [21, 6, 90, 2126], "dtype": "uint8", "packing": "packbits" },
  "data": { "file": "rip.bin", "byte_count": 0, "crc32": "0x..." }
}
```

The array shape is `(passes, channels, nozzles, columns)`, so it varies with
the head: `(P, 6, 90, W)` on `c6n90`, `(P, 4, 60, W)` on `c4n180`. A payload
with no `head_layout` block is read as `c6n90`, which keeps older payloads
loadable.

## Printer job: `job.json` + `job.bin`

`encoder.py` writes this pair for the daemon. The binary stores 147-byte lines
consecutively, in pass order. The header indexes them with per-pass
`line_count` values and contiguous offsets.

```jsonc
// job.json  (sibling: job.bin)
{
  "format_version": "0.1.0",
  "geometry_fingerprint": "0x108865B9",   // the FRAME; must match the firmware
  "head_name": "c6n90",                   // WHICH HEAD; checked host-side
  "head_fingerprint": "0x9C244CD5",
  "dpi": 630, "bytes_per_line": 147, "total_passes": 42,
  "print_width_mm": 49.7, "print_height_mm": 229.0, "padded_width_mm": 60.5,
  // one entry per plumbed ink (per the encoder's ink map)
  "channel_offsets_px": { "M": 0, "C": 23, "LC": 174, "LM": 197, "Y": 348, "K": 371 },
  "passes": [ { "y_position_mm": 0.0, "y_delta_mm": 0.04, "line_count": 0 } ],
  "data": { "file": "job.bin", "byte_count": 0, "line_count": 0, "crc32": "0x..." }
}
```

The container is defined once, in `paintress-protocol`'s `paintress_job`
module, and vendored by the encoder and the daemon.

## The two fingerprints

A job has separate fingerprints for its frame and head. Both are CRC-32 values
calculated from canonical strings by the `paintress-protocol` generator.

| | Covers | Written as | Checked against | A change means |
|---|---|---|---|---|
| **Frame** | line size, clocks, edges, packing, PSRAM line size | `geometry_fingerprint` | the firmware's `FRAME_HASH`, reported at `IDENTIFY` as `profile_hash` | reflash |
| **Head** | head name, nozzle count, channel list | `head_name` + `head_fingerprint` | the daemon's `--head` | re-RIP |

The frame fingerprint describes the data sent by the firmware. Heads sharing
that frame can use the same binary. The daemon checks the job's head against
its machine configuration because the board cannot detect the physical head.

## Key sizes

| Constant | Value | Meaning |
|---|---|---|
| `bytes_per_line` (USB line) | 147 | Packed column on the wire and in the job. See [Printhead data protocol](../concepts/printhead-protocol.md) |
| PSRAM line | 196 | The same column expanded to 49 32-bit words for DMA |
| `FRAME_HASH` | `0x108865B9` | Frame fingerprint, shared by both shipped heads |
| `head_fingerprint` | `0x9C244CD5` (`c6n90`), `0x93255450` (`c4n180`) | Which head a job was packed for |
