# Building and testing

## Firmware

Build the firmware with the Raspberry Pi Pico SDK and `PICO_SDK_PATH` set:

```sh
cd paintress-firmware
mkdir build && cd build
cmake .. && make -j          # -> paintress_firmware.uf2
```

Flash through BOOTSEL mass storage or `picotool load -v -x`. To put a running
board into USB boot mode without pressing its button, run `picotool reboot -u
-f` before loading.

Tests for framing, dispatch, line packing, slot states and safety logic run
with a host C compiler. They do not require the Pico SDK:

```sh
cmake -S tests -B tests/build
cmake --build tests/build
ctest --test-dir tests/build --output-on-failure
```

With a board on USB there are two smoke tests:

```sh
python hw_tests/fw_smoke.py <port>      # IDENTIFY / command round-trip
python hw_tests/swath_test.py <port>    # full swath receive path (non-firing)
```

## Daemon

Daemon tests run as Python scripts or under pytest, without hardware:

```sh
cd paintress-daemon
for t in tests/test_*.py; do python "$t"; done
```

The tests use simulated dependencies to cover streaming, latched faults, abort
behaviour, serial reconnection, command validation and event reporting.

## RIP and encoder

```sh
cd paintress-rip-encoder
for t in tests/test_*.py; do python "$t"; done
```

Golden tests compare the encoder's output bytes across several ink maps and
DPI values. Investigate any changed output before updating the expected files.
Regenerate them only when the format change is intentional, and explain why in
the commit.

## Protocol and head profiles

```sh
cd paintress-protocol
make check        # generated bindings match the schema and profiles
make test         # round-trip + job + profile + C<->Python goldens
make sync-check   # every vendored copy is byte-identical
```

Run `make sync-check` before a release to catch consumer bindings that differ
from the generated originals.

## Klipper plugin

The plugin runs inside Klipper. Check syntax with `python -m py_compile
extras/*.py` and run the tests that do not need its reactor:

```sh
cd paintress-klipper-extras
for t in tests/test_*.py; do python "$t"; done
```

These tests cover travel bounds and both origin modes (`test_positioning.py`),
client reconnection (`test_client_reconnect.py`) and event polling
(`test_event_inbox.py`). Full integration testing requires Klipper, the daemon
and a controller board.

## This site

```sh
pip install -r requirements-docs.txt
mkdocs serve            # live preview
mkdocs build --strict   # link-checking build (CI gate)
```
