# Documentation index

This specification covers the HomematicIP (HmIP) protocol as spoken
between a host and a UART-attached radio coprocessor module, and the
cloud key-exchange step used during device pairing. See the top-level
[README](../README.md) for the project's scope, legal notice, and
licensing — this page is purely a reading guide to the `docs/` folder.

## Conventions used in this specification

- **Multi-byte integers are big-endian** wherever byte order is stated
  explicitly — the serial frame envelope's `Length` and `CRC` fields
  (`serial-protocol.md`), the Inclusion Accept frame's MAC sequence
  number, and the CCM*-style construction's length field and keystream
  block index (both `security-and-inclusion.md`). Fields whose byte order
  is never stated (e.g. the Inclusion Request frame's manufacturer code
  and device-type fields) should not be assumed one way or the other —
  treat them as opaque unless a source confirms otherwise.
- **Bit numbering is MSB-first, per byte**: bit 7 is the most significant
  bit, bit 0 the least, matching the `bit(byte, N)` / `bits(byte, hi..lo)`
  pseudocode helpers used throughout `mac-application-frames.md`. The
  `[Field:width]` bracket notation used for every header byte in this
  spec (MAC header, Application header, Status frame flags) lists fields
  in that same order, left to right — most significant field first.
- **Hex notation is not uniform across this spec, and this document does
  not silently normalize it** — two different, non-conflicting styles are
  in active use, each accurate where it appears:
  - Named protocol constants (e.g. `0xFD` sync byte, `0xFFFF` CRC init in
    `serial-protocol.md`) and the cloud key-exchange JSON API's binary
    fields (explicitly specified as "uppercase hexadecimal, no
    separators" in `security-and-inclusion.md`) both use **uppercase**
    hex digits.
  - The spec's detailed worked byte-level examples (the captured Status
    and Answer frames in `mac-application-frames.md`/`walkthrough.md`,
    and the Inclusion Request/Get-inclusion-data/Inclusion Accept/
    Configuration-write examples in `security-and-inclusion.md`/
    `configuration-frames.md`) instead use **lowercase**, bare (no `0x`
    prefix) hex, with single-byte fields space-separated from their
    neighbors and multi-byte fields (addresses, SecurityNumber,
    IntegrityCode) written as one contiguous run with no internal spaces.
  - These two styles genuinely disagree on digit case; that's noted here
    rather than resolved one way. `docs/walkthrough.md` follows the
    worked-example style, since it reuses real bytes from elsewhere.

## Suggested reading order

Each layer builds on the one before it — read in this order the first
time through. "Missing" in the Confidence column is what's not yet
wire-verified.

| File | Description | Confidence |
|---|---|---|
| [`serial-protocol.md`](serial-protocol.md) | The transport envelope: how bytes on the wire are framed, checksummed, and sequenced between a host and the coprocessor module. Everything else is carried inside this layer. | Mostly verified on the wire — missing: serial commands 21–30 (OTAU, adapter MIC/connection-data/inclusion-accept-data variants), documented in name only |
| [`mac-application-frames.md`](mac-application-frames.md) | The structure of the actual HmIP protocol data unit (the "MAC frame") carried inside that envelope, and the Application-layer header carried inside *that*. Read this before anything below, since every later document references MAC/Application header fields by name. | Mostly verified on the wire — missing: FrameType values outside the named table, the Time Info "current time" response sub-type's byte value |
| [`security-and-inclusion.md`](security-and-inclusion.md) | How a frame's security fields (introduced above) are actually used, and the device-pairing handshake that establishes them, including the cloud key-exchange service. This is the protocol's "bootstrapping" phase — everything below assumes a device has already been paired. | Mostly verified on the wire — missing: the cloud key exchange's JSON payload (shape only, no captured example), local-only (non-cloud) key exchange end-to-end |
| [`native-device-links.md`](native-device-links.md) | One of two frame families built on top of the ordinary Application frame: device-to-device links, created and read back via a specific Application FrameType. | Mostly verified on the wire — missing: multi-radiator groups, link-refusal behavior, removal edge cases |
| [`configuration-frames.md`](configuration-frames.md) | The other frame family built on the Application frame: reading and writing a device's own settings, including the weekly heating-program encoding. Shares its request/response envelope with the link mechanism above (both are sub-types of the same Configuration FrameType). | Mixed — missing: most per-setting addresses beyond what's been directly captured, boost-mode's full byte layout, the mode/window-state/active-profile/valve-adaption commands |
| [`device-catalog.md`](device-catalog.md) | A short, independent reference: how to map the numeric manufacturer/device-type fields from an Inclusion Request frame to a real model name. Can be read any time after `security-and-inclusion.md`; it doesn't depend on the two frame-family files above. | Its "Verified table" entries are wire-verified against real hardware; it deliberately does not reproduce the full vendor catalog — see the file itself |
| [`glossary.md`](glossary.md) | An alphabetical reference for protocol-specific terms and acronyms used throughout the documents above. Not meant to be read start-to-end; look up a term as you encounter it. | N/A |
| [`walkthrough.md`](walkthrough.md) | A worked trace of a device inclusion through to ordinary post-inclusion traffic, tying the transport/frame/security layers together as one continuous example: real captured bytes for every step except the cloud key-exchange call itself (structural there). Best read after `security-and-inclusion.md`, as a synthesis of what came before rather than new reference material. | N/A |
| [`known-gaps.md`](known-gaps.md) | Read last: a consolidated list of what remains unconfirmed across every document above, framed as an invitation for community verification rather than a project TODO list. | N/A |

## Cross-references between files

The documents are not fully self-contained — some topics are described
in one file and relied on, without re-explanation, in another:

- `security-and-inclusion.md` uses the MAC header's `SecurityControl`
  field and the Application header's `ResponseRequested`/`StayAwake`
  fields exactly as defined in `mac-application-frames.md`, and the
  Send protocol frame/Get inclusion data/Add link partner serial
  commands exactly as defined in `serial-protocol.md`.
- `native-device-links.md` and `configuration-frames.md` both build on
  the Application FrameType mechanism from `mac-application-frames.md`
  (they are, respectively, FrameType 1's own sub-family and a
  closely-related one), and both depend on `serial-protocol.md`'s
  transmission discipline and `security-and-inclusion.md`'s
  listener-mode/wake-window concept for reaching battery-powered
  devices reliably.
- `configuration-frames.md`'s "self" link-partner address sentinel is
  the same field `native-device-links.md` uses for genuine
  device-to-device links — reading one without the other risks confusing
  "configuring a device's own settings" with "configuring a specific
  link's parameters."
