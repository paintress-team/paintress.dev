# Troubleshooting

Use the console or daemon log to match an error to the tables below. Each
entry lists the likely cause and the next checks to make.

## Print failures

| Symptom | Likely cause | Action |
|---|---|---|
| `Swath N was swept but the firmware never reported PRINT_COMPLETE` | The start trigger edge never reached the board | Check the trigger wiring (`trigger_output_pin` to the board's trigger input, shared ground) and that `trigger_distance` is shorter than the sweep. `PAINTRESS_TEST_TRIGGER` with a scope or an LED confirms the edge. |
| `trigger_timeout` fault | The firmware armed and gave up waiting for that same edge | Same checks. It waits about 10 s before giving up. |
| `serial_lost` fault mid-print with the board still plugged in, and a "watchdog reboot" message after reconnecting | The watchdog rebooted an unresponsive core; the resulting USB drop cancelled the print | `reconnect`, or just retry the print, since the plugin's abort/reset path reopens the link. Report repeated watchdog resets as a firmware issue. |
| `dac_latched` NACK on arm or purge | The DAC was latched off by an abort, or by the thermal monitor if enforcement is enabled, and no reset followed | Run the daemon `reset`, or retry `PAINTRESS_PRINT`, since the pipeline restart reboots a latched board. Check the daemon log for `SAFETY:` messages and head temperature. |
| `profile_mismatch` on `PAINTRESS_OPEN_PAYLOAD` | The job's frame (line size, clocks, packing) does not match the connected firmware build | Re-encode the job against the current protocol, or reflash the firmware. The two frame fingerprints have to agree. |
| `head_mismatch` on `PAINTRESS_OPEN_PAYLOAD` | The job was packed for a different printhead than the daemon's `--head`. The board cannot detect this, so the daemon is the only place it can be caught | The error message identifies both heads. Re-RIP for the fitted head (`rip.py --head ...`), or restart the daemon with the head that is actually on the machine. Keep `--head` matched to the physical head. |
| `invalid_line_delay` | `print_speed` × payload DPI gives an interval outside 1–65535 µs | Raise `print_speed`, or use a higher-DPI payload. |
| The print fails immediately with an old fault | A fault latched at the end of the previous print | `abort`/reset clears it, and retrying `PAINTRESS_PRINT` also starts clean. |

## Connection issues

| Symptom | Likely cause | Action |
|---|---|---|
| `device_not_connected` | The serial link is not open | `PAINTRESS_CONNECT_SERIAL` first, and check `serial_port`. |
| `serial_lost` event or fault | USB to the board dropped mid-session, which cancels the print | The daemon never reconnects on its own. Run `reconnect` (or `reset`/`abort`) to restore the link, then start the print over. Check the cable and the power. |
| Wire-protocol mismatch on connect | Firmware and daemon from different protocol generations | Update both sides together. The daemon refuses a major-version mismatch by design. |
| `timeout` on a command | The watchdog expired and the daemon cannot cancel the worker | The command may still finish in the background, so check `PAINTRESS_STATUS` before retrying anything that changes state. |

## Print quality

| Symptom | Likely cause | Action |
|---|---|---|
| Image far too dark, and worse at high DPI | Uncalibrated tone response; drop overlap grows with DPI | Calibrate the RIP's ink limits and tone curves for your DPI. See [RIP and halftoning](../concepts/rip-and-halftoning.md). |
| Regular thin horizontal gaps | A dead or clogged nozzle, one missing row per pass | `PAINTRESS_PURGE` the affected channel and repeat. If the gaps remain after purging, print `rip.py --target nozzle_check`, read the failed nozzles off the numbered comb, record them with `tools/nozzle_check_from_scan.py --dead ...`, and RIP with `--nozzle-comp retouch`. |
| Colour fringing along the sweep direction, every colour offset by the same amount | The encoder's column gaps do not match the head's real geometry | Print `rip.py --target col_align`, then `tools/col_align_from_scan.py` prints calibrated `--column-gap` and `--group-gap` values for the encoder. Calibrate once per head. |
| A pale line repeating at a fixed interval down the print | A band seam, the Y advance between bands | Turn on band feathering with `rip.py --band-overlap 12`. Seams recur once per band step, so they show up three times as often on `c4n180` as on `c6n90`. |
| The first columns of every pass are compressed | Firing started before the head was at constant speed | Increase `trigger_distance` to at least v²/2a. The plugin warns about this. |
| Passes shifted horizontally against each other | Trigger edge jitter, or a wiring problem | Make sure the edge comes from the motion MCU pin rather than a host-timed source, and check `PAINTRESS_TEST_TRIGGER` for repeatability. |

## Status LED (controller board)

| Blink | Meaning |
|---|---|
| Fast, about 50 ms | Printing |
| Medium, about 150 ms | Receiving swath data |
| About 250 ms | USB not connected |
| Slow, about 500 ms | Idle, connected |
| Very fast, forever, at boot | PSRAM initialisation failed. The board is unusable until it is power-cycled or repaired |

## Getting more information

`PAINTRESS_STATUS` shows the daemon status, including the job state and any
latched `print_fault` with its root cause.

Use the daemon console for detailed diagnosis. It timestamps commands,
responses, firmware events and `LOG` lines. Firmware logs are also sent to TCP
clients as `firmware_log` messages.