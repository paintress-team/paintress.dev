# Serial protocol

The daemon and firmware exchange framed binary messages over USB CDC. This
page lists the frame layout, commands, events and compatibility checks.

!!! note "Single source of truth"
    The wire codes and payload structs are generated from the
    `paintress-protocol` schema and vendored into the firmware
    (`paintress_protocol.h`) and the daemon (`paintress_protocol.py`).
    Change the schema and regenerate; never edit a wire value on one side.
    See [Changing the protocol](../dev/changing-the-protocol.md).

## Frame layout

```
  [START 0xAA] [LEN_LO] [LEN_HI] [TYPE] [PAYLOAD ...] [CRC_LO] [CRC_HI]
```

Frames use CRC-16-CCITT with an initial value of `0xFFFF`, stored
little-endian and calculated over the header and payload. `MAX_PAYLOAD_SIZE`
is 256. After a framing error, both sides scan byte by byte to resynchronise.

| Type | Value | Carries |
|---|---|---|
| `COMMAND` | `0x01` | command byte plus parameters (host to firmware) |
| `DATA` | `0x02` | one packed 147-byte column line |
| `ACK` | `0x03` | echoed command byte plus optional response data |
| `NACK` | `0x04` | echoed command byte plus error code |
| `EVENT` | `0x05` | event byte plus data (firmware to host, unsolicited) |
| `LOG` | `0x06` | level byte plus UTF-8 text, the firmware's only logging |

## Commands (host to firmware)

| Code | Name | Parameters | Notes |
|---|---|---|---|
| `0x01` | `BEGIN_SWATH` | `swath_id(2)`, `line_count(4)` | Allocates a slot. The ACK carries `swath_id(2)`, `slot(1)`. |
| `0x02` | `END_SWATH` | — | ACK carries `swath_id(2)`, `complete(1)`. An incomplete swath is discarded. |
| `0x03` | `ARM` | `swath_id(2)`, `line_delay_us(2)` | Arms a received swath. It fires on the hardware start trigger and then clocks one column per `line_delay_us`. Each ARM supplies its own timing interval. |
| `0x04` | `ABORT` | — | Electrical kill-switch. The DAC is de-energised and latched off; nothing else is interrupted, and a firing swath runs dry to its end with no ink leaving the head. While latched, `ARM` and `PURGE` are NACKed `DAC_LATCHED`. |
| `0x05` | `GET_STATUS` | — | ACK carries 17 bytes of status: both slots, `receiving`, `printing`, `dac_latched`. |
| `0x06` | `RESET` | — | A chip reboot. The firmware ACKs, drains the ACK, then reboots through the watchdog. USB drops and re-enumerates in a second or two; the host waits for this expected reconnect. Rebooting clears slots, the DAC latch and all other state. |
| `0x07` | `PURGE` | `channel(1)`, `pulses(1)` | Fires one channel's column `pulses` times. The channel is bounds-checked. |
| `0x0C` | `IDENTIFY` | — | ACK carries `wire_id(2)`, `profile_hash(4)`, `fw_build(4)`, `boot_flags(1)`: the version and geometry handshake. In `boot_flags`, bit 0 means this boot came from a watchdog reboot, a commanded `RESET` included, and bit 1 means a genuine wedge recovery where the watchdog fired on its own. |

`0x08` and `0x09` (`SET_TIMING`, `GET_TIMING`), `0x0A` (`SET_HEAD_POWER`)
and `0x0B` (`SET_DAC_POWER`) are retired and not reused.

## Events (firmware to host)

| Code | Name | Payload | Meaning |
|---|---|---|---|
| `0x01` | `PRINT_STARTED` | `swath_id(2)` | The start edge was seen and the swath is firing. |
| `0x02` | `PRINT_COMPLETE` | `swath_id(2)` | The swath fired to its last line. |
| `0x03` | `PRINT_ERROR` | `swath_id(2)`, `error_code(1)` | Reports `error_code = DAC_LATCHED` when a queued `ARM` is rejected at power-up after a kill (`ABORT`, `RESET`, safety). No ink has fired for that arm. A swath loop already firing continues to its end. |
| `0x04` | `TRIGGER_TIMEOUT` | `swath_id(2)` | Armed, but no start edge arrived within the guard window. |

`PRINT_COMPLETE`, `PRINT_ERROR` and `TRIGGER_TIMEOUT` each free the swath's
slot.

## Error codes (NACK and PRINT_ERROR)

`OK(0x00)`, `UNKNOWN_CMD(0x01)`, `INVALID_PARAM(0x02)`,
`NO_SLOT_AVAILABLE(0x03)`, `SWATH_NOT_FOUND(0x04)`, `SWATH_NOT_READY(0x05)`,
`PRINT_IN_PROGRESS(0x06)`, `SLOT_BUSY(0x07)`, `LINE_COUNT_MISMATCH(0x08)`,
`VERSION_MISMATCH(0x09)`, `PROFILE_MISMATCH(0x0A)`, `DAC_LATCHED(0x0C)`.
`0x0B` (`PRINT_ABORTED`) is retired, since no path interrupts a firing swath
any more.

## Handshake

On connect the daemon sends `IDENTIFY` and refuses the session if the
firmware's `wire_id` major version differs from its own.

`profile_hash` identifies the frame layout sent by the firmware: line size,
clocks, edges and packing. Heads sharing that frame use the same binary.
Because the board cannot identify the fitted head, the daemon checks both the
frame and the configured head when loading a job:

| Check | Against | Failure |
|---|---|---|
| job's `geometry_fingerprint` = firmware's `profile_hash` | the connected firmware | `profile_mismatch` |
| job's `head_fingerprint` = the daemon's `--head` | the machine's configuration | `head_mismatch` |

Both checks must pass before the job can print.