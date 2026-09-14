# Daemon TCP API

The daemon uses NDJSON, one JSON object per line, on TCP port 9000. Clients
receive responses matched to their requests and asynchronous messages. The
Klipper plugin's `paintressd_client.py` is the reference client.

## Commands

```json
{"cmd": "load_job", "id": 1, "filepath": "/path/to/job.json"}
```

```json
{"type": "response", "cmd": "load_job", "id": 1, "success": true}
```

| Group | Commands |
|---|---|
| Serial link | `connect` (`port`), `disconnect` (`transport_only`), `reconnect` |
| Job | `load_job` (`filepath`), `unload_job`, `job_info` |
| Status | `status` (daemon-level, includes any latched `print_fault`), `get_status` (firmware slots and `dac_latched`), `identify` (including `boot_flags`) |
| Printing | `print` (`swath_id`, `line_delay_us` in 1–65535; blocks until streamed, then arms, and the firing interval travels with each print), `abort` (firmware kill-switch and reboot), `reset` (firmware reboot, returns once the board is back) |
| Control | `purge` (`channel`, `pulses`) |

`send_swath`, `set_timing`, `get_timing` and `dac_power` are retired. The
pipeline handles streaming, each `print` carries its timing interval, and the
engine controls the DAC. For direct serial debugging, use the firmware
repository's `hw_tests/` tools.

Failed commands return `success: false`, an `error` code and a readable
`message`. Common codes are listed below.

| Code | Meaning |
|---|---|
| `device_not_connected`, `not_connected`, `no_port` | no serial link |
| `profile_mismatch` | the job's frame does not match the connected firmware |
| `head_mismatch` | the job was packed for a different printhead than the daemon's `--head` |
| `no_job_loaded`, `swath_not_found`, `swath_out_of_sequence` | job or state |
| `invalid_line_delay`, `invalid_channel`, `invalid_pulses` | parameter range |
| `serial_lost` | the link dropped unexpectedly and the print is cancelled |
| `timeout`, `no_response`, `internal_error` | see the note below |

A `head_mismatch` message identifies the job's head and the daemon's
configured head. Re-RIP for the physical head, or correct the daemon's
`--head` setting if that setting is wrong.

!!! note "`timeout` is a watchdog, not a cancel"
    The daemon cannot interrupt a running command thread. A `timeout`
    response means the command may still finish in the background, with its
    result discarded and logged. Check `status` before retrying anything
    that changes state.

## Asynchronous messages

Read every incoming line: asynchronous messages can arrive between command
responses.

| `type` | When | Key fields |
|---|---|---|
| `welcome` | On connect | `version`, `device_connected`, `serial_port`, `job_loaded` |
| `event` | Firmware events and daemon faults | `event` (`print_started`, `print_complete`, `print_error`, `trigger_timeout`, `pipeline_error`, `serial_lost`), `swath_id`, `data` (raw payload as hex), `error` (code name, for `print_error`) |
| `firmware_log` | A firmware `LOG` frame | `text` |
| `keepalive` | Every 30 s | `timestamp` |
| `error` | An invalid client line | `error` |

## Failure semantics

The daemon latches faults from firmware errors, trigger timeouts, streaming
failures and serial loss. Subsequent `print` requests report the stored cause
immediately. `load_job`, `abort` and `reset` clear the fault.

`reset` and `abort` reboot the firmware. The daemon expects the resulting USB
drop and waits for the board to re-enumerate before responding. An unexpected
drop cancels the print and latches and broadcasts `serial_lost`. Recovery
requires an explicit `reconnect`, `reset` or `abort`; the daemon does not
reconnect automatically.

More detail on the [daemon component page](../components/daemon.md) and in
[Troubleshooting](../guide/troubleshooting.md).
