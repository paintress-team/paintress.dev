# Glossary

| Term            | Meaning                                                                       |
|-----------------|-------------------------------------------------------------------------------|
| Swath / pass    | One horizontal strip printed in a single printhead sweep.                     |
| Band            | One block of consecutive image rows, covered by `passes_per_band` interleaved passes. The head advances by the shortest plumbed slot rather than by the column length. |
| Column          | One vertical line of nozzle firings at a single X position.                   |
| Channel         | One ink colour. The head profile defines the channels: C, M, Y, K, LC, LM on `c6n90`, and C, M, Y, K on `c4n180`. |
| DPI             | Dots per inch; must be a multiple of the head's nozzle pitch.             |
| Nozzle pitch    | Spacing of nozzles along Y, in nozzles per inch (90 npi or 180 npi).           |
| Nozzle count    | Number of nozzles used for printing in a slot; this may be less than its physical count. |
| Head profile    | The YAML description of one printhead in `paintress-protocol/profiles/`, from which every consumer's constants are generated. |
| Slot (head)     | One ink feed on the head: a contiguous run of nozzles at a known place.       |
| Ink map         | Ink connected to each slot, read left to right facing the head. This records the machine's plumbing. |
| Bus             | Data carried by one pin: 180 two-bit nozzle codes and a 32-bit window map, totalling 392 bits. |
| Lead-in         | Passes below the image that let stacked ink slots reach the first rows. Their scheduled Y positions are negative. |
| Halftoning      | Turning continuous-tone channel values into a 1-bit fire/skip pattern.       |
| Dot gain        | Increase in printed dot area as ink spreads on the substrate. The RIP compensates before halftoning.  |
| Wire format     | The packed 147-byte line layout sent over USB and stored in PSRAM as 196 B.   |
| Window selector | Hardware that selects and sequences the nozzle firing windows.                      |
| Purge           | Firing nozzles to clear or prime ink before printing.                         |
| Firing grid     | Fixed-period schedule the firmware uses to fire one column at a time.         |
| Slot (PSRAM)    | One half of the firmware's double buffer, holding a single swath.             |
| Overscan        | Distance travelled before and after the print area so the head is at speed.       |
| RIP             | Raster image processor, `rip.py`.                                            |
| Payload         | A loaded `.json` job, in the Klipper plugin's vocabulary.                     |
| Waveform        | The shaped voltage pulse that deforms a piezo actuator to eject a drop.       |
| Firing window   | A time slot in the head's firing cycle; sequenced by the window selector.     |
| Window-enable map | 32-bit word shifted in with each data block, mapping nozzle codes to the firing windows in which they receive the waveform. |
| Trigger distance | X travelled into a sweep before the start edge is emitted by the motion MCU. |
| Frame fingerprint | CRC-32 of what the firmware shifts onto pins (line size, clocks, edges, packing). Checked against the firmware at `IDENTIFY`; changing it means a reflash. Written into a job as `geometry_fingerprint`. |
| Head fingerprint | CRC-32 of which head a job was packed for (name, nozzle count, channels). Checked host-side by the daemon against its `--head`; changing it means a re-RIP, not a reflash. |
