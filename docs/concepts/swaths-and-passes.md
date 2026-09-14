# Swaths and passes

A swath, or pass, is one horizontal strip printed in a single sweep of the
head. The RIP divides an image into these passes and records their positions
in the job.

## Bands and interleaving

A head's nozzles sit at a fixed pitch along Y, 1/90″ on `c6n90` and 1/180″
on `c4n180`. Anything finer than that is built by interleaving passes, each
offset from the last by one image row:

```
passes_per_band = dpi / nozzle_pitch_npi
lines_per_band  = band_step_nozzles * passes_per_band
```

At 630 dpi on a `c6n90` that works out at 7 passes per band. Each pass
prints every 7th row, offset from the previous one by 1/630″.

!!! warning "It is the pitch, not the nozzle count"
    The two happen to coincide on `c6n90` (90 nozzles at 90 npi) and do not
    on `c4n180` (60 nozzles fired at 180 npi). DPI must be a multiple of the
    pitch. `python rip/rip.py --list-dpi --head <name>` prints the values
    that qualify.

## The band step is the shortest plumbed slot

A band contains `lines_per_band` consecutive image rows covered by
`passes_per_band` passes. The head advances between bands by the height of the
shortest plumbed slot. This lets every ink cover the image without leaving
gaps.

On `c6n90`, each slot covers a full column, so the band step is one inch. On
`c4n180`, the shortest ink block has 60 nozzles in a 180-nozzle column. Its
band step is one third of an inch, although the column is a full inch long.

The RIP does this slicing, giving each pass the rows its nozzles will cover,
and the job header records every pass's absolute Y position
(`y_positions_mm`) along with the delta to the next one.

## Lead-in: why the first passes sit below the image

Heads with slots at different heights need lead-in passes below the image so
the highest slot can reach the bottom rows. The first Y positions are
therefore negative: about −16.93 mm on `c4n180`, or two band steps. Only some
inks fire during the lead-in; the final band steps complete coverage for the
others.

The plugin uses the job's pass schedule to calculate travel bounds, including
this movement below the origin. If the lower Y bound fails preflight, move the
print origin up. Do not use `extra_margin` to add lead-in clearance: it
changes where the image lands.

## Consequences worth knowing

Each printed row corresponds to one nozzle and pass. A failed nozzle therefore
produces regularly spaced missing rows, which makes it possible to identify it
from a nozzle-check print.

Band seams repeat at the band step. A 1/3″ step produces three times as many
seams as a 1″ step. Use `--band-overlap` to feather the boundaries.

The plugin follows each pass's Y position and sweeps X. The job contains the
band geometry, so the plugin does not need head-specific pass calculations.

## Overscan and the travel rectangle

The head must reach constant speed before firing. `start_overscan` provides
acceleration distance before the print area; the trigger fires
`trigger_distance` into the sweep. `end_overscan` provides space after the
print area and must cover deceleration at `print_speed`.

The plugin folds both, plus the configured head offsets, into a travel
rectangle (`print_bounds`) and checks it against the machine's axis limits
before printing.

## Padded width

Nozzle columns are offset along X. The encoder shifts their data to compensate
and pads the image width so every ink can reach the full image. The plugin
uses this padded width from the job metadata when calculating bounds.