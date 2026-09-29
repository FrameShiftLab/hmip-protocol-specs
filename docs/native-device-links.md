# Native device-to-device links

HmIP devices can be linked directly to one another ("Verknüpfung" in the
vendor's own terminology) so that, for example, a window sensor reporting
"open" causes a linked thermostat to show a window-open indicator and
adjust its heating — entirely device-to-device, with no host/server
involvement at runtime once the link exists. This document covers how
such links are created, removed, and read back.

## Which channels may be linked

Each device exposes one or more numbered channels, and each channel has a
declared link role (sender or receiver) and one or more link types. Two
channels may be linked only if their roles differ (one sender, one
receiver) and they share at least one link type. This is a general
mechanism — the same rule governs every link family below, not just
window-sensor links.

### Window-sensor → thermostat link

**Verified on the wire.** A window/door sensor's channel that reports open/
closed state acts as the sender; a thermostat exposes a dedicated
"heating shutter-contact receiver" channel as the target. Concretely, for
the device families observed:

- Window/door sensor devices: sender is **channel 1**.
- Thermostat devices (both wall-mounted room thermostats and radiator
  thermostats): receiver is **channel 4**.

(A thermostat's climate channel, typically channel 1, is a *different*
channel from the link-receiver channel above — once linked, the reported
window state surfaces as a regular state value on the thermostat's own
climate channel, not on the link-receiver channel itself.)

## Creating a link

**Verified on the wire** for the window-sensor↔thermostat case; the
general mechanism below is confirmed to extend correctly to other device
pairs (see "Heating-group links" below), though not every individual
byte sequence has been captured for every device combination.

Creating a link sends **one frame to each of the two devices** being
linked — a Configuration-family frame (Application FrameType 1, see
`configuration-frames.md`), sub-type "create link":

```
payload = [own channel][0x01][partner address (3 bytes)][partner channel][0x00][partner's own listener mode]
```

- `[own channel]` is the sending device's own channel number in this pair.
- `[partner address]` is the plain 3-byte MAC address of the other device
  in the link (no address-type marker byte for a plain device address).
- `[partner channel]` is the partner's channel number in this pair.
- The trailing byte is the **partner's own listener mode** (the same value
  originally declared in that device's inclusion request) — each side of
  a link needs to know how to reach the other.

**Application header for this frame:** response requested, FrameType
Configuration (1). **StayAwake depends on the target device's own
listener mode**: a true **event listener** (no periodic listening at
all — only reachable right after *it* transmits; typically
battery-powered sensors, see `serial-protocol.md`'s listener-mode table)
must receive this frame with StayAwake set, and the frame can only
actually be delivered during that device's own brief wake window (see
"Sleeping devices" below). Any **cyclic or burst listener** — which
includes both wall thermostats (real example: `HmIP-WTH-2`, declared
mode 3) *and* radiator thermostats (real example: `HmIP-eTRV-2`,
declared mode 11, genuinely cyclic) — receives it with StayAwake clear,
sent immediately with the ordinary burst-mode transmission its declared
listener mode calls for (`serial-protocol.md`); no host-side
wait-for-transmission synchronization is needed for either of these two
real device types, despite `HmIP-eTRV-2`'s mode genuinely involving a
periodic sleep cycle — the module's own repeated burst transmission
covers that without host involvement (see `serial-protocol.md`'s
listener-mode table for why this differs from the true event-listener
case).

**Both frames are answered with an ordinary Answer/ACK** (see
`mac-application-frames.md`). A link is considered established once both
halves have been acknowledged; if one side is not immediately reachable
(a sleeping sensor), the create-link request for that side is held and
sent as soon as that device is next heard from — while the request is
pending, the other, already-linked side is fully functional.

**No configuration-parameter write follows a default-profile link
creation** — link creation alone is sufficient for the link to take
effect; per-link parameter customization (e.g. non-default thresholds) is
a separate, subsequent configuration write addressed to the real link
partner (see `configuration-frames.md`), only sent when a
non-default value is actually wanted.

### Constructing a link's two frames

Every link operation below is symmetric: build one payload addressed to
device A (pointing at device B) and one addressed to device B (pointing
at device A), send each to its own device. `listener_mode` here is the
value declared in that device's own inclusion request (see
`security-and-inclusion.md`) — needed by the *other* side's frame, to
pick the right `StayAwake`/burst mode handling, not by the frame's own
sender.

```
function create_link_frames(device_a, device_b) -> (frame_for_a, frame_for_b):
    # device_a / device_b each have: address, channel, listener_mode

    payload_for_a = [device_a.channel, 0x01] + device_b.address + [device_b.channel, 0x00, device_b.listener_mode]
    payload_for_b = [device_b.channel, 0x01] + device_a.address + [device_a.channel, 0x00, device_a.listener_mode]

    frame_for_a = configuration_frame(payload_for_a, response_requested=true,
                                       stay_awake = is_event_listener(device_a.listener_mode))
    frame_for_b = configuration_frame(payload_for_b, response_requested=true,
                                       stay_awake = is_event_listener(device_b.listener_mode))
    return (frame_for_a, frame_for_b)

function remove_link_frames(device_a, device_b) -> (frame_for_a, frame_for_b):
    payload_for_a = [device_a.channel, 0x02] + device_b.address + [device_b.channel, 0x00]
    payload_for_b = [device_b.channel, 0x02] + device_a.address + [device_a.channel, 0x00]
    return (
        configuration_frame(payload_for_a, response_requested=true, stay_awake=is_event_listener(device_a.listener_mode)),
        configuration_frame(payload_for_b, response_requested=true, stay_awake=is_event_listener(device_b.listener_mode)),
    )
```

Sending each resulting frame still goes through the ordinary "one frame
in flight per device, wait for its Answer/ACK" discipline (see
`serial-protocol.md`) and, for an event-listener device, the wake-window
mechanism described below — `create_link_frames`/`remove_link_frames`
above only build the two payloads, they don't imply anything about
scheduling their delivery.

### Worked example, verified on the wire — a real window-sensor → radiator-thermostat link

A real `HmIP-SWDO`'s channel 1 linked to a real `HmIP-eTRV-2`'s channel
4 (a window-sensor → radiator-thermostat shutter-contact-receiver link),
frame addressed to the eTRV-2 (the receiver side of this specific
half — link creation sends one frame per side, this is the one seen
first in the capture):

```
04 01 773a08 01 00 04
```

- `04` — own channel (the eTRV-2's shutter-contact-receiver channel).
- `01` — request type (create link).
- `773a08` — partner address: the real SWDO's own short address for
  this session.
- `01` — partner channel (the SWDO's sender channel).
- `00` — the fixed trailing byte before the listener-mode byte.
- `04` — the SWDO's own declared listener mode. This is a genuinely new
  value: `serial-protocol.md`'s listener-mode table only lists real
  observations for `1`, `3`, `9`, `11` — `4` for a real `HmIP-SWDO` is a
  new data point for that table (not yet cross-referenced against that
  device's own Inclusion Request to confirm which named category it
  falls under; added here as a raw observation).

Application header `0x81` (response requested, StayAwake clear,
FrameType Configuration) — matches "Creating a link" above (a
cyclic/burst-listening thermostat, not an event-listening sensor, on the
*receiving* end of this particular half). Answered with an ordinary
Answer/ACK, `code = 0x00` (OK).

### Worked example, verified on the wire — real link removal

The reverse of the link above (a different `HmIP-SWDO`'s channel 1,
removed from the same `HmIP-eTRV-2` channel 4), frame addressed to the
eTRV-2:

```
04 02 6e6e0d 01 00
```

- `04` — own channel (unchanged from the create-link example).
- `02` — request type (remove link).
- `6e6e0d` — partner address (the second SWDO's own short address for
  this session — a different physical device from the create-link
  example above).
- `01` — partner channel.
- `00` — trailing byte (no listener-mode byte on removal, confirming
  `remove_link_frames`'s payload shape above against a real capture).

Application header `0x81`, same as create-link. Answered with an
ordinary Answer/ACK, `code = 0x00` (OK) — the device's Answer arrived
with the *same* application sequence number as the remove-link request,
confirming correlation the same way an ordinary Configuration write
does.

## Removing a link

Same addressing, sub-type "remove link", one frame to each device:

```
payload = [own channel][0x02][partner address (3 bytes)][partner channel][0x00]
```

(No trailing listener-mode byte on removal — see `remove_link_frames`
above; **verified on the wire**, see the worked example below.)

## Reading links back

A device's currently-linked partners for a given channel can be queried:

```
request:  [channel][0x03][start index]
response: [channel][0x0a][last partner number]
          followed by (address: 3 bytes, partner channel: 1 byte) entries, repeated
```

A device also proactively reports when a link partner fails to answer it:
`[channel][0x0e][partner address (3 bytes)]`.

Reading the current link list before creating a new link is useful for
idempotency (avoid re-sending a create-link request for a pair that
already exists — sending it again anyway is harmless, real vendor
software logs it but does not treat it as an error) and for verifying a
link took effect after creation.

## Sleeping devices — the real constraint

Some device classes (typified by battery-powered window/door sensors) are
**event listeners**: their radio receiver is only active for a short
window immediately after *they* transmit something, not on any fixed
schedule the access point could otherwise predict. Any frame destined for
such a device — a link-creation frame, a configuration write, an ordinary
acknowledgement — can only be delivered during one of these brief windows,
by:

1. Waiting for the device to send something (any status update).
2. Answering that transmission's own acknowledgement with the StayAwake
   flag set.
3. Sending the queued frame(s) immediately within that same window, one
   at a time, waiting for each one's own Answer/ACK before sending the
   next (see `serial-protocol.md`'s transmission discipline).

An access point implementation that does not implement this — that
instead relies on a fixed polling interval to flush queued outgoing work —
will reliably miss this class of device's receive window and silently
fail to deliver time-sensitive frames to it, including link-creation
frames and configuration writes, even though everything else about the
frame is byte-correct. This was directly observed as a root cause of
"device shows link/command not confirmed" symptoms that persisted despite
otherwise-correct protocol implementation.

## Heating-group links (wall thermostat ↔ radiator thermostat)

**Derived from source analysis of vendor server software** (the specific
link-pair definitions are not present in any of the publicly shipped,
non-compiled configuration data available for analysis — they were
recovered from server-side bytecode); **subsequently confirmed to work
correctly on real hardware in practice** (multiple rooms, each with one
wall thermostat and one radiator thermostat plus window sensors, all
links reported as answered/established, and the paired devices observed
behaving correctly as a heating group — a wall thermostat's setpoint
change is reflected on its linked radiator thermostat's valve). An exact
byte-level capture of vendor software performing this specific link
family has not been independently taken; the confirmation is behavioral
(the derived frames work), not a byte-for-byte comparison against a
reference capture.

### Worked example, verified on the wire — a real heating-group link pair

A real `HmIP-WTH-2` channel 1 linked to a real `HmIP-eTRV-2` channel 6,
both frames from the same `addLink` call (one real link operation,
role-matched by the devices' own declared `LINK_SOURCE_ROLES`/
`LINK_TARGET_ROLES` — both sides declare the role `CLIMATE_CONTROL_WTH_TRV`
on these specific channels, confirmed via a live channel-description
query before sending):

```
frame to eTRV-2 (channel 6):  06 01 7df915 01 00 03
frame to WTH-2  (channel 1):  01 01 3878f0 06 00 0b
```

- Frame to eTRV-2: own channel `06`, partner address `7df915` (the
  WTH-2's own short address), partner channel `01`, listener mode `03`
  — matches `serial-protocol.md`'s table (`HmIP-WTH-2`, mode 3, triple
  burst) exactly.
- Frame to WTH-2: own channel `01`, partner address `3878f0` (the
  eTRV-2's own short address), partner channel `06`, listener mode `0b`
  (11 decimal) — matches the same table's `HmIP-eTRV-2` entry (mode 11,
  cyclic + triple burst) exactly.
- Both answered with an ordinary Answer/ACK, `code = 0x00`.

**This example only settles one link pair, and it doesn't cleanly map
onto the 3-row table below.** The table's second row describes "wall
thermostat's climate-transceiver ↔ radiator's *climate-receiver*
channel" — channel 2 on a real `HmIP-eTRV-2`. What was actually observed
here links to channel **6** instead (a channel independently confirmed,
via the same live channel-description query, to declare the *specific*
role `CLIMATE_CONTROL_WTH_TRV`, distinct from channel 2's generic
`CLIMATE_CONTROL` role) — and only **one** `addLink` call was made for
this heating group, not three. Either the table below is derived
slightly wrong about which channel this particular pair actually uses,
or a real heating group only needs this one role-matched pair rather
than the three rows below (with the other two only mattering for some
condition not exercised here, e.g. multiple radiators). This capture
can't distinguish between those two explanations — it directly confirms
one real, working link pair, and directly raises a question about the
table below that wasn't there before.

A "heating group" links one wall thermostat with up to 8 radiator
thermostats (and up to 8 window sensors, using the window-sensor link
above, extended to every thermostat in the group). Forming a two-device
heating group (one wall thermostat, one radiator thermostat) creates
**three separate link pairs**, each following the same create-link
mechanism as above (one frame to each device per pair, so 3 frames to
each device in total):

| Pair | Purpose (approximate) |
|---|---|
| Radiator's own climate-transceiver channel ↔ wall thermostat's climate-receiver channel | radiator reports its measured temperature to the wall thermostat |
| Wall thermostat's own climate-transceiver channel ↔ radiator's climate-receiver channel | wall thermostat pushes room temperature/setpoint to the radiator |
| Wall thermostat's secondary transmitter channel ↔ radiator's corresponding receiver channel | an additional climate-control link the vendor's group logic always creates alongside the two above |

Both halves of the affected devices are cyclic/burst listeners in the
observed device classes (`HmIP-WTH-2` mode 3, `HmIP-eTRV-2` mode 11 —
see `serial-protocol.md`'s listener-mode table), so **none of the three
pairs' frames need the event-listener wait-for-transmission mechanism**
— they're sent with the ordinary burst-mode transmission their declared
listener mode calls for, without waiting for a wake window, but the same
"one frame in flight per device, wait for its Answer before the next"
discipline still applies. (`HmIP-eTRV-2`'s mode does involve a genuine
periodic sleep cycle, unlike `HmIP-WTH-2`'s — see the note in "Creating
a link" above — but that's handled by burst-mode transmission alone, not
by anything the frame sequence here needs to account for.)

**With more than one radiator thermostat in a group**, the radiators are
additionally linked to each other (one climate-transceiver-to-receiver
pair per ordered pair of radiators) — this specific detail is inferred
from the general pattern and has not been independently confirmed.

**Not yet confirmed:** the exact wire byte sequence for heating-group
link removal in practice (expected to mirror the window-sensor removal
pattern above), and whether removing the last radiator thermostat from a
group also removes the wall thermostat's own links, or leaves them
dangling until explicitly removed.
