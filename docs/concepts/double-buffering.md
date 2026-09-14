# Double buffering

The daemon prepares swaths ahead of printing, using two firmware slots so data
transfer can overlap with the current pass.

## The firmware slots

The board divides its 8 MB of external PSRAM into two slots, each holding one
complete swath. The host sends `BEGIN_SWATH`, one `DATA` frame per 147-byte
column, then `END_SWATH`. On arrival, the firmware expands each line to 196
bytes (49 32-bit words), ready for DMA. No unpacking is needed during firing.

A slot moves through `EMPTY → RECEIVING → READY → PRINTING → EMPTY`. If a
transfer contains fewer lines than declared, the firmware discards it. Only a
complete swath can be armed. One slot can receive data while the other prints.

## The daemon's pipeline

The daemon holds the full job in memory. A background worker keeps the
firmware slots supplied with swaths.

On `load_job`, or when connecting with a job already loaded, the daemon clears
stale slots and starts sending the first two swaths. A `print` request waits
until its swath is ready, then sends `ARM`. Each `PRINT_COMPLETE` frees a slot
for another transfer. If the next swath is already buffered, it can be armed
without waiting for USB.

Streaming runs in the daemon because a swath transfer can take seconds. Doing
that work in Klipper's reactor would block motion planning and can cause
"Timer too close" errors. The plugin only moves, arms, sweeps and confirms
completion. See [Design rationale](../dev/design-rationale.md).

## When things fail

A streaming failure, trigger timeout, firmware error or serial loss latches a
fault in the daemon. The next `print` reports that fault immediately.

`ABORT` immediately disables and latches off the DAC. A running swath loop
finishes without firing ink, and further `ARM` or `PURGE` requests return
`DAC_LATCHED`. The daemon follows with `RESET`, which clears the latch and
both slots. Once recovery succeeds, the pipeline can restart without reloading
the job.

An interrupted print restarts from its first swath; partial resume is not
supported. See [Troubleshooting](../guide/troubleshooting.md) for fault
recovery.