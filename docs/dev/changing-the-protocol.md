# Changing the protocol

Edit wire definitions in `paintress-protocol/schema/protocol.yaml` and head
definitions in `profiles/*.yaml`. These generate the C and Python bindings
copied into the firmware, daemon and encoder. Update the sources and
regenerate; do not edit generated copies.

## The workflow

1. **Edit the schema**: `schema/protocol.yaml`, which holds the enums,
   commands, events, error codes and fixed-offset payload structs.
2. **Bump the version.** Additive, backward-compatible changes bump MINOR;
   breaking ones bump MAJOR. The handshake constant follows,
   `wire_id = (MAJOR << 8) | MINOR`. The daemon refuses a firmware whose
   major differs, so a MAJOR bump means both sides ship together.
3. **Regenerate** with `make gen`. It writes four files: the wire bindings
   (`generated/c/paintress_protocol.h`,
   `generated/python/paintress_protocol.py`) and the head bindings
   (`generated/c/head_config.gen.h`, `generated/python/head_profiles.py`).
4. **Verify.** `make check` confirms the generated files are in sync with
   the sources. `make test` runs the round-trip, job and profile tests plus
   the cross-language golden vectors, where Python packs the expected bytes
   and C `memcmp`s them. `make compile-check` builds the header under the
   real ARM toolchain so that every `_Static_assert` fires.
5. **Distribute.** `make sync` copies the canonical bindings into the repos
   that vendor them, and `make sync-check` fails if any vendored copy has
   drifted. Run it in CI and before a release.
6. **Update the consumers**, meaning firmware dispatch and usb_link, daemon
   handlers and parsers, and the command tables in their READMEs.
7. **Record it** in `CHANGELOG.md`.

## Changing a head profile

Use the same generation and verification steps for head profiles. The field
being changed determines which compatibility check is affected:

| Field changed | Fingerprint moved | Consequence |
|---|---|---|
| `nozzle_count`, `channels`, `name` | head | re-RIP the jobs, no reflash |
| `bytes_per_line`, `edges`, `clocks`, `packing`, `psram_line_size` | frame | reflash *and* re-RIP |
| `slot_inks`, gaps, `base_dpi`, `window_enable_map` | neither | nothing; host-side reference data |

A new head also needs a per-bus nozzle mapping in `encoder/head_packer.py` and
a layout in `rip/head_layout.py`. These describe how the head is packed and
arranged.

## Rules of thumb

Keep retired codes reserved. For example, command `0x0A` must not be
reassigned.

Payload structs use flat, little-endian fields at fixed offsets, with no
optional fields or nested length prefixes. The protocol treats raster data as
opaque bytes.

The profile defines the DATA line size, recorded as `bytes_per_line` in the
job header and included in the frame fingerprint. The protocol transports each
line without interpreting its contents.