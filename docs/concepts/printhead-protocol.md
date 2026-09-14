# Printhead data protocol

The encoder packs each printed column into the format described here. The
firmware stores and sends it to the head's shift registers. This interface was
reverse-engineered.

Both supported heads use this format. Their per-bus tables assign different
physical nozzles to the same data positions.

## The bus is the unit

Each data pin carries one bus: 180 nozzle positions with a 2-bit code each,
followed by a 32-bit window-enable map. That is 392 bits, sent at two bits per
clock.

The two bits of a code are not adjacent. A bus is two blocks of 180:

```
 index   0 .......... 179 | 180 .......... 359 | 360 ...... 391
         first bit of the | fire bit of the    | window-enable
         code, positions  | code, positions    | map
         0..179           | 0..179             | (32 bits)
```

Bus position `p` has its code's first bit at index `p` and its fire bit at
`180 + p`. Only codes `00` and `01` are emitted today, so the first block is
always zero.

| Code | Meaning today | Reserved for |
|---|---|---|
| `00` | do not fire | |
| `01` | fire | |
| `10`, `11` | *(unused)* | |

The window-enable map determines when each nozzle code receives the drive
waveform.

## What rides a bus is the head-specific part

A table of 180 entries per bus maps data positions to physical nozzles.

On `c6n90`, each bus interleaves two 90-nozzle columns. Three buses cover all
six columns: `A` carries slots 0 and 1, `B` carries slots 2 and 3, and `C`
carries slots 4 and 5.

On `c4n180`, each bus serves one 180-nozzle column with a direct mapping, `p →
nozzle p`. Each 180-bit block divides into three runs of 60. Only two buses
are wired; the third sends zeros.

The tables differ; the packing code does not
(`paintress-rip-encoder/encoder/head_packer.py`).

## The window-enable map

The last 32 bits of each bus select the firing windows for each nozzle code.
This map must be measured on the physical head; geometry alone does not
determine it. The supported heads use different maps:

| Head | Window map | Meaning |
|---|---|---|
| `c6n90` | `00000000000000100000000000000000` | bit 14 |
| `c4n180` | `00000000000000000000000000000010` | bit 30, window 7, code `01` |

Both maps enable only code `01`, giving one drop size: `01` fires the waveform
and `00` does not fire.

## Clocking and the 147-byte line

The buses shift simultaneously over a 3-line DDR data bus, so every clock
carries 6 bits, three pins on the rising edge and the same three on the
falling edge. A 392-bit bus takes 196 clocks, and one complete column is:

```
196 clocks × 6 bits = 1176 bits = 147 bytes
```

Unused buses send zeros. This lets `c4n180` and `c6n90` share the 147-byte
frame and the same firmware binary.

Each 147-byte column is one USB `DATA` frame and one job-file line
(`bytes_per_line`). The firmware expands it into 49 32-bit words with 24
payload bits per word, then stores those 196 bytes in PSRAM for PIO/DMA.
Packing saves 25% on USB transfer size, and expanding on receipt avoids
repacking during firing.

## Sequencing

For each column, the data bus loads the nozzle data, the window selector
latches it and sequences the firing windows, and the DAC sends the waveform to
enabled nozzles. Loading the next column's data overlaps the current firing
interval.

Line size, clocks, edges and packing are included in the frame fingerprint
exchanged at `IDENTIFY`. Changing them requires a firmware reflash. Nozzle
count and channels belong to the separate head fingerprint, checked by the
daemon. See [File
formats](../reference/file-formats.md).