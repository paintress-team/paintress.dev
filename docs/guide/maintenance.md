# Maintenance

!!! warning "Work in progress"
    Procedures are being documented and checked on the reference build.

## Purging

Use a purge to fire nozzles without printing an image. This helps clear air or
dried ink and prime the head:

```gcode
PAINTRESS_PURGE CHANNEL=all PULSES=20
PAINTRESS_PURGE CHANNEL=magenta PULSES=50
```

Move the head over a waste area or wiper before purging. Drops land directly
below its current position. Purging is also the first check for missing-nozzle
streaks; see [Troubleshooting](troubleshooting.md).

## Nozzle health

The RIP can generate a labelled nozzle check for visual inspection without a
scanner. Each plumbed ink gets a comb pattern: one dash per nozzle, arranged
diagonally, with a numbered ruler and guides every 10 nozzles.

```sh
python rip/rip.py --target nozzle_check -o nz.json --dpi 630 --preview nz.png
# encode and print nz.json, then read it by eye:
#   find the gap -> follow the guide line up to the ruler -> read the number
python tools/nozzle_check_from_scan.py --dead "C:89;M:4,7;K:58" \
       --profile rip/profiles/mymedia.json
```

Several consecutive missing nozzles usually indicate a clog. Purge and reprint
the check before recording failures. `rip.py --nozzle-comp retouch` can
compensate for isolated failed nozzles and the edges of a cluster. It cannot
fill the interior of a cluster, where no healthy neighbour is available.

Failed nozzles are recorded by physical slot, so the record remains valid when
the ink map changes.

## Storage and idling

**TBD:** capping and parking recommendations, ink shelf-life, and the
first-use priming procedure for the reference head.
