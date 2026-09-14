# Daemon

`paintress-daemon` connects the Klipper host to the firmware. This Python
service loads print jobs, manages USB serial communication and keeps the
firmware supplied with swath data.

## Responsibilities

- Serving a line-delimited JSON (NDJSON) protocol over TCP, port `9000` by
  default.
- Loading a job (`.json` header plus `.bin` file) and holding its swaths
  in memory. It refuses a job whose frame does not match the connected
  firmware, and one whose head does not match the printhead the daemon was
  started with (`--head`).
- Managing the USB serial connection and the framed binary protocol.
  Per-line frame encoding is accumulated into 64 KB writes.
- Owning the double-buffer pipeline: keeping both firmware slots fed and
  streaming the next swath while the current one prints.
- Forwarding firmware events and `LOG` frames to every connected client
  asynchronously.

## Internals

An `asyncio` TCP server handles clients, backed by serial threads. Commands
that change state, such as `connect`, `load_job`, `print` and `reset`, run in
one `cmd-worker` thread to avoid races. Read-only commands (`status`,
`identify`, `job_info`) use a thread pool, so they can respond while a slower
command runs. `SerialManager` manages the port and separate RX and TX threads.

Urgent `abort` and `reset` frames bypass the normal TX queue, so they do not
wait behind a long swath transfer.

### Failure model

An unexpected serial loss cancels the print. The daemon latches the fault,
stops streaming and waits for an explicit `reconnect`, `reset` or `abort`; it
does not retry automatically. The interrupted 2D job must restart from its
first swath. The Klipper plugin handles the error so an enclosing 3D print can
continue.

## Running

```sh
python -m paintress_daemon --head c4n180
```

`--head` selects the fitted printhead. Other options are `--tcp-port` (default
9000), `--host` (default loopback), `--baud` and `--debug`.

## TCP API

The API uses one JSON object per line. Each `response` includes the request's
`id`. The daemon also sends asynchronous `welcome`, `event`, `firmware_log`,
`keepalive` and `error` messages. See the [Daemon TCP
API](../reference/tcp-api.md) for commands and message formats.