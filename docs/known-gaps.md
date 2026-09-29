# Known gaps

An honest, consolidated list of what this specification does **not** yet
have confirmed with real hardware, or leaves genuinely open. If you have
access to real HmIP devices and can help settle any of these, that
verification is exactly the kind of contribution this project needs —
please open an issue or pull request with what you observed (raw byte
captures welcome).

## MAC and Application frames (`mac-application-frames.md`)

- **Application FrameType values not listed in the Frame types table**
  (i.e. every value except the ones named there) haven't been observed
  or named in this spec — a coverage gap in the table, not a claim about
  what those values do or don't do.
- **Time Info "current time" response sub-type's own byte value** hasn't
  been independently captured — only the "request" sub-type's value
  (`0x01`) is wire-verified, from real device-originated traffic.

## Security and inclusion (`security-and-inclusion.md`)

- **Local-only key exchange**: the CCM*-style cryptographic construction
  is understood and self-consistent (round-trips correctly against
  itself), but has not been confirmed end-to-end against a real device
  completing inclusion via this path alone. This spec's real, wire-verified
  inclusion capture (Inclusion Request through Inclusion Accept) doesn't
  settle this either — nothing observable at the UART layer distinguishes
  a cloud-mode inclusion from a local-only one, so which path that
  specific capture used isn't determinable from the trace alone.
- **The Inclusion Request frame's joining-device one-time-key
  contribution field** (see `security-and-inclusion.md`'s field table)
  appears unused by every inclusion-handling code path traced so far —
  its actual purpose is unclear.
- **Serial command 20** ("set encrypted network key") is observed on the
  wire, real payload and all, but this spec doesn't have a confirmed
  field-level breakdown of that payload — see
  `security-and-inclusion.md`'s Get-inclusion-data section.
- **Device removal (exclusion)**: the exact payload contents of the two
  NetworkManagement frames (frame-type bytes `f0`/`f2`) in the removal
  sequence, beyond their single frame-type byte, are unconfirmed — as is
  whether a non-sleeping device skips the transmission-pending queueing a
  sleeping device requires, and whether removing a device that's still a
  member of a native link tears those links down automatically. (The
  three-frame sequence's overall shape is wire-verified — see
  `security-and-inclusion.md`'s "Device removal" section.)

## Native device links (`native-device-links.md`)

- **Heating-group link definitions** (wall thermostat ↔ radiator
  thermostat, and radiator ↔ radiator): a real captured link pair used
  `HmIP-eTRV-2` channel 6 (role `CLIMATE_CONTROL_WTH_TRV`), not channel 2
  as derived vendor-bytecode analysis suggested for the second row of a
  3-row table (see `native-device-links.md`'s heating-group worked
  example). Whether the derived table is wrong about the channel, or a
  real heating group genuinely only needs this one role-matched pair
  (with the table's other two rows only mattering under some condition
  not exercised in this capture, e.g. multiple radiators in the group),
  is not yet determined.
- **Radiator-to-radiator link pairing** in a group with more than one
  radiator thermostat (whether every ordered pair is linked, or some
  other topology) is inferred from the general link-role pattern, not
  confirmed.
- **Heating-group link removal**: the exact behavior when removing a
  member (whether removing the last radiator also removes the wall
  thermostat's own links, or leaves them dangling) is not confirmed.
- **Link refusal behavior**: what a device's Answer/response looks like
  when it refuses a link (versus simply not answering) has not been
  observed.
- Whether a link partner's *side* of a link must be created in a specific
  order (does a receiver reject a link if its own create-link request
  arrives before the sender's?) is unconfirmed.

## Configuration frames (`configuration-frames.md`)

- **Reading a full weekly heating program back** (paging through all
  13–14-entry chunks needed to cover a full 182-byte program) has only
  been partially exercised (the first two chunks of one real read were
  observed) — the full paging loop to completion, and a re-read
  immediately after writing the same program slot, are both unconfirmed.
- **Per-setting (list, index, bit) addresses** are known for only a
  handful of settings on a handful of device types, confirmed either by
  a live read or by cross-referencing vendor device-description data.
  The great majority of device types and settings have not been
  individually confirmed.
- **Direct Execution Command byte layouts**: boost-mode's value byte is
  only partially understood (a 1-bit difference between enable/disable
  observed, the rest of that byte's bits unconfirmed — see
  `configuration-frames.md`'s Direct Execution Command section). Mode
  (auto/manual), window-state, active-profile, and valve-adaption Direct
  Execution Commands remain entirely uncaptured — still derived from
  source analysis only. (Separately, an independent reimplementation
  project reports having *sent* several of these commands to a real
  device and observed the expected resulting state change — see that
  project's own notes — which is corroborating evidence but not a
  byte-for-byte reference capture from vendor software itself.)
- Whether returning a thermostat to "automatic" mode in practice uses a
  dedicated control-mode field versus the ordinary setpoint frame's own
  mode bits, and how a temporary manual override interacts with
  automatic mode, are not fully confirmed.
- The OTAU (over-the-air firmware update), remote live update, and OEM
  container frame families are entirely uninvestigated.

## General

- This specification covers a coprocessor-attached access-point
  implementation talking to end devices. It does not cover routed
  (multi-hop) traffic, IPv4/IPv6 address extensions, or the classic
  (non-IP) "BidCoS-RF" sub-1GHz protocol family that some of the same
  hardware also supports — these are out of scope.
- Firmware-version reporting by a device is understood (a packed version
  field in its inclusion request); whether and how an access point can
  determine that a *newer* firmware version is available, versus simply
  the device's current version, has not been investigated here.
