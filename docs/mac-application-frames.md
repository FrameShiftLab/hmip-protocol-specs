# MAC and Application frame formats

**Status: verified on the wire** unless noted. This is the structure of
the bytes carried as the "MAC frame" payload inside a serial
Send protocol frame / Received event command (see
`serial-protocol.md`) — the actual HmIP protocol data unit.

## MAC header

Every MAC frame starts with a MAC header, minimum 9 bytes:

```
Byte 0: [Fragmentation:1][Cyclic:1][SecurityControl:2][ContentType:4]
Byte 1: [reserved:1][IPAddressExtension:4][reserved:2][PiggybackAck:1]
Byte 2: [HomematicIPVersion:4][LocalHopLimit:3][QosLow:1]
Bytes 3-5: MAC source address (3 bytes)
Bytes 6-8: MAC destination address (3 bytes)
[+4 bytes SecurityNumber, +4 bytes IntegrityCode  — only if SecurityControl == ENABLED]
[+4 bytes SecurityNumber, +8 bytes IntegrityCode  — only if SecurityControl == ENABLED_2]
[+N bytes IP extension data — only if IPAddressExtension != SINGLE_HOP (0)]
[+1 byte cyclic telegram counter — only if Cyclic bit is set]
```

Bit layout of byte 0, most significant bit first: fragmentation flag,
cyclic-message flag, 2-bit security control, 4-bit content type.

```
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|F|C|Sec| CType |
+-+-+-+-+-+-+-+-+

F     = Fragmentation (1 bit)
C     = Cyclic (1 bit)
Sec   = SecurityControl (2 bits)
CType = ContentType (4 bits)
```

Leftmost = most significant bit, matching the prose above. The ruler
above numbers positions left to right starting at 0 (the classic
RFC-diagram convention: position 0 = leftmost = the MSB) — the *reverse*
of how this spec's own pseudocode numbers bits (`bit(b0, N)`/
`bits(b0, hi..lo)` below number bit 7 as the MSB, bit 0 as the LSB). Both
describe the same 8 bits; don't conflate the diagram's position 0 with
the pseudocode's bit 0 when cross-referencing the two. Field boundaries
above are a visual restatement of the table below, not a new claim.

**ContentType** values (byte 0, low nibble):

| Value | Meaning |
|---|---|
| 0 | HmIP Application — carries an Application frame (see below) |
| 1 | HmIP Network Management — device inclusion and related handshake frames |
| 3 | HmIP Route Management |
| 4 | HmIP MAC Control |
| 2, 5–9 | compressed/general IP variants, not otherwise covered here |

**SecurityControl** values (byte 0, bits 4-5):

| Value | Meaning |
|---|---|
| 0 | Disabled — no security fields present, header is exactly 9 bytes |
| 1 | Enabled — 4-byte SecurityNumber + 4-byte IntegrityCode (MIC) follow the addresses |
| 2 | Enabled (extended) — 4-byte SecurityNumber + 8-byte IntegrityCode |
| 3 | Reserved |

**Non-obvious, verified finding:** a real access-point implementation sets
the security-ENABLED flag on essentially every post-inclusion frame it
sends to a device — but transmits the SecurityNumber and IntegrityCode
fields **as all zero bytes**, or in some observations omits them from the
logical frame it hands to the transmission layer entirely, relying on the
**coprocessor module itself** to compute and insert the real
message-integrity code before the frame actually goes out over RF. A host
implementation building frames "by the book" (computing and inserting a
real MIC itself) is not what real vendor software does for ordinary
application traffic — the module is trusted to do this transparently for
frames flagged as secured, given a network key it already possesses,
established during that device's own inclusion (see
`security-and-inclusion.md`).

**Address extension / cyclic fields**: not exercised in normal single-hop
traffic between an access point and a directly-reachable device; present
for completeness, largely out of scope for a typical access-point
implementation.

### MAC header: decode and encode

**The `SecurityControl` byte alone does not tell a decoder how many
security bytes actually follow.** Per the finding above, an
access-point-transmitted frame can have `SecurityControl=ENABLED` with
its SecurityNumber/IntegrityCode bytes *entirely absent* (0 bytes, not
4+4) — the real worked Answer frame in
[`walkthrough.md` § Step 6](walkthrough.md#step-6--access-point-answers)
is exactly this case (9-byte header, straight into the Application
header, despite `SecurityControl=ENABLED`). A real captured
*device*-originated frame, by contrast, always carries its full
SecurityNumber/IntegrityCode (see
[`mac-application-frames.md`'s own worked example](#worked-example-a-real-captured-status-frame)
and `security-and-inclusion.md`'s Inclusion Accept worked example's
follow-on traffic). Unconditionally consuming 4+4 (or 4+8) bytes
whenever `SecurityControl != DISABLED` — as an earlier version of this
pseudocode did — misparses that Answer-frame case, silently consuming
the *next* field's bytes (the Application header) as if they were
security fields. A decoder needs the transport-level context (is this
data the host itself is constructing to send outward, where it may have
chosen to omit the fields, or is this data received off the wire, where
they're always present?) — it can't be recovered from `SecurityControl`
alone:

```
function decode_mac_header(data, security_fields_expected = true) -> MacHeader:
    # security_fields_expected: true for a frame received off the wire
    # (Received event) — security fields are always present there in
    # practice. Pass false only when decoding a frame the host itself
    # constructed for outgoing transmission (Send protocol frame) and
    # is known to have omitted the fields for, per the finding above —
    # this can't be derived from `data` itself.
    b0 = data[0]
    fragmentation    = bit(b0, 7)
    cyclic_message   = bit(b0, 6)
    security_control = bits(b0, 5..4)     # 2-bit field
    content_type     = bits(b0, 3..0)     # 4-bit field

    b1 = data[1]
    ip_address_extension = bits(b1, 6..3)  # 4-bit field
    piggyback_ack         = bit(b1, 0)

    b2 = data[2]
    homematic_ip_version = bits(b2, 7..4)
    local_hop_limit       = bits(b2, 3..1)
    mac_qos_low             = bit(b2, 0)

    idx = 3
    mac_source      = data[idx : idx+3]; idx += 3
    mac_destination  = data[idx : idx+3]; idx += 3

    security_number = integrity_code = null
    if security_fields_expected and security_control == ENABLED:
        security_number, idx = take(data, idx, 4)
        integrity_code,  idx = take(data, idx, 4)
    else if security_fields_expected and security_control == ENABLED_2:
        security_number, idx = take(data, idx, 4)
        integrity_code,  idx = take(data, idx, 8)
    # else: SecurityControl says ENABLED, but the fields are known (out
    # of band) to have been omitted for this outgoing frame — nothing to
    # consume here; the coprocessor inserts the real ones before RF TX

    if ip_address_extension != SINGLE_HOP:
        ip_extension_extra, idx = take(data, idx, length_for(ip_address_extension))

    if cyclic_message:
        cyclic_telegram_counter = data[idx]; idx += 1

    return MacHeader(..., header_length = idx)   # header_length: where the payload starts

function encode_mac_header(h) -> bytes:
    b0 = (h.content_type & 0x0F)
    b0 |= (h.security_control & 0x03) << 4
    b0 |= (1 << 6) if h.cyclic_message else 0
    b0 |= (1 << 7) if h.fragmentation else 0

    b1 = (h.ip_address_extension & 0x0F) << 3
    b1 |= 1 if h.piggyback_ack else 0

    b2 = (h.homematic_ip_version & 0x0F) << 4
    b2 |= (h.local_hop_limit & 0x07) << 1
    b2 |= 1 if h.mac_qos_low else 0

    out = [b0, b1, b2] + h.mac_source + h.mac_destination
    if h.security_control in (ENABLED, ENABLED_2) and h.security_fields_present:
        out += h.security_number + h.integrity_code   # all-zero placeholder bytes, if present at all — see the finding above; a real AP implementation may omit this branch's bytes entirely for outgoing frames and still set the flag
    if h.ip_address_extension != SINGLE_HOP:
        out += h.ip_extension_extra
    if h.cyclic_message:
        out += [h.cyclic_telegram_counter]
    return bytes(out)
```

### Application header: decode and encode

```
function decode_application_header(data, mac_header) -> ApplicationHeader:
    idx = mac_header.header_length
    b = data[idx]
    response_requested          = bit(b, 7)
    stay_awake                   = bit(b, 6)
    frame_type                    = bits(b, 5..0)      # 6-bit field
    application_sequence_number  = data[idx + 1]
    payload                       = data[idx + 2 :]
    return ApplicationHeader(...)

function encode_application_header(h) -> bytes:
    b = h.frame_type & 0x3F
    b |= (1 << 7) if h.response_requested else 0
    b |= (1 << 6) if h.stay_awake else 0
    return encode_mac_header(h.mac_header) + bytes([b, h.application_sequence_number & 0xFF]) + h.payload
```

### Worked example: a real captured Status frame

The following 37-byte MAC frame was captured on the wire, re-verified
directly against the raw serial trace (as the payload of a serial
Received event command, itself preceded on the wire by that command's
own RSSI byte, `0x39` — the coprocessor's own signal-strength
measurement; see the two-RSSI-values note below, since that byte is
serial-protocol-level, not part of the MAC frame itself). Byte-by-byte:

```
10 01 8e  2cf049  b1235a  00000067 1052057c  85 02  04 80 01 01 00 de 43 03 00 1a 04 01 68 00 00 0d 01 00
│  │  │   │       │       │        │         │  │  └─────────────────────────── Application payload (status data, opaque here) ────────────────────────┘
│  │  │   │       │       │        │         │  └─ application sequence number = 2
│  │  │   │       │       │        │         └─ Application header byte 0 = 0x85 = ResponseRequested(1) | StayAwake(0) | FrameType=5 (Status)
│  │  │   │       │       │        └─ IntegrityCode (real, non-zero value here — see note below)
│  │  │   │       │       └─ SecurityNumber (real, non-zero value here — see note below)
│  │  │   │       └─ MAC destination address (the access point)
│  │  │   └─ MAC source address (the reporting device)
│  │  └─ byte 2 = 0x8e = HomematicIPVersion=8, LocalHopLimit=7, QosLow=0
│  └─ byte 1 = 0x01 = no IP address extension, PiggybackAck=1
└─ byte 0 = 0x10 = Fragmentation(0) Cyclic(0) SecurityControl=01(ENABLED) ContentType=0000(Application)
```

**Note on the security fields here.** Unlike the "sent as all-zero, or
omitted entirely — the coprocessor fills in the real MIC" finding
described above, this frame is a **device → access point** frame (a
Received Event), not one the access point transmits — that finding is
specifically about outgoing, access-point-sent frames. Here, the
coprocessor hands the host a real, non-zero SecurityNumber and
IntegrityCode: `00000067` / `1052057c`. This spec doesn't decode the
SecurityNumber's own semantics (e.g. whether/how it's used as an
anti-replay counter) — treat it as an opaque field for now.

An earlier version of this example (still visible in this repository's
history) mis-transcribed this frame — wrong source address, an all-zero
security field that doesn't match either this frame's real bytes or the
finding it cited, and two wrong payload bytes. The bytes above are
re-verified directly against the raw serial capture.

The Status frame's own payload (everything after the Application header)
starts `04 80 ...`: flags byte `0x04` (only the `Booted` bit set here,
`ContentFormat=00` uncompressed), RSSI byte `0x80` — the "no reading"
sentinel described below, not a real measurement — followed by
uncompressed status data entries, which this specification deliberately
does not decode further (see above).

## Application header (ContentType = Application)

When a MAC frame's content type is Application, its payload begins with a
2-byte Application header immediately after the MAC header:

```
Byte 0: [ResponseRequested:1][StayAwake:1][FrameType:6]
Byte 1: Application sequence number (u8)
[remaining bytes: frame-type-specific payload]
```

Byte 0's bit layout (same ruler convention as the MAC header diagram
above — position 0 = leftmost = MSB, the reverse of this spec's own
`bit(b, N)` pseudocode numbering; see the note there):

```
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|R|S| FrameType |
+-+-+-+-+-+-+-+-+

R = ResponseRequested (1 bit)
S = StayAwake (1 bit)
FrameType = 6 bits
```

- **ResponseRequested**: the sender expects an acknowledgement (see
  "Answer frames" below).
- **StayAwake**: a hint to a battery-powered, cyclic-listening device that
  the sender has more work queued for it and it should keep its receiver
  open a little longer than its normal cycle. Real access-point software
  sets this on the acknowledgement to a device's status update whenever a
  configuration write is pending for that device — this is how a host
  implementation keeps a brief-listen-window device reachable long enough
  to deliver a multi-frame configuration write (see
  `configuration-frames.md`) within the same wake period.
- **FrameType** (6 bits): see the table below.
- **Application sequence number**: increments per frame from a given
  source; a device answers with the same sequence number in its
  acknowledgement.

### Frame types

| Value | Name |
|---|---|
| 1 | Configuration (see `configuration-frames.md`) |
| 2 | Answer (acknowledgement) |
| 5 | Status |
| 6 | Direct Execution Command (see `configuration-frames.md`) |
| 8 | Unconditional Switch Command |
| 9 | Conditional Switch Command |
| 10 | Level Command |
| 14 | Heating Controller Mode |
| 16 | OEM Container |
| 32 | Service |
| 33 | Status Request |
| 34 | Trigger Asynchronous Status |
| 35 | Time Info |
| 62 | Remote Live Update |
| 63 | OTAU Command |

Values not listed above haven't been observed or named in this
specification — that's a gap in this table's coverage, not a claim that
those values are unused or invalid.

### Answer frames

A frame with `ResponseRequested` set and addressed to the access point
itself must be acknowledged with an Answer frame (FrameType 2):
Application header (FrameType=2, `ResponseRequested=false`,
`StayAwake` = true only if a follow-up write such as a configuration
write is pending for that device, sequence number = the request's own
sequence number) followed by a single payload byte: the answer type
(`0` = ACK). **This applies regardless of whether the sender actually
understands or intends to act on the request's payload** — a device that
receives a command it cannot or will not apply still acknowledges it; a
successful acknowledgement is not evidence the command took effect
(confirm the resulting device state separately, e.g. via its next Status
frame).

### Time Info frames

A device periodically requests the current time (FrameType 35,
sub-type "request" — **verified on the wire: sub-type byte `0x01`**,
observed identically from two different real devices; the "current
time" response sub-type's own byte value hasn't been independently
captured — see `known-gaps.md`). The request payload encodes the requesting device's
own standard-time UTC offset and a full daylight-saving-time transition
rule (week-of-month/month/day-of-week/time-of-day for both the "spring
forward" and "fall back" transitions, each in quarter-hour units) — the
device supplies its own DST rule in every request; an access point has no
region-specific DST logic of its own and simply applies the supplied rule
to the current UTC time to compute the local time to answer with. The
response (sub-type "current time") carries the resulting local
date/time fields. Devices retry this request until it is answered;
answering it correctly is one of two independent conditions (the other
being ordinary Answer-frame acknowledgement above) for a device to
consider itself in good contact with its access point.

### Status frames

A Status frame (FrameType 5) reports device state. Its payload:

```
Byte 0: flags — [LowBattery:1][ConfigPending:1][PiggybackAppAck:1][DutyCycle:1][reserved:1][Booted:1][ContentFormat:2]
Byte 1: RSSI value (signed 8-bit)
[remaining bytes: a sequence of status data entries, see below]
```

Byte 0's bit layout (same ruler convention as the diagrams above —
position 0 = leftmost = MSB, the reverse of this spec's own pseudocode
bit numbering; see the note under the MAC header diagram):

```
 0 1 2 3 4 5 6 7
+-+-+-+-+-+-+-+-+
|L|C|P|D|-|B| CF|
+-+-+-+-+-+-+-+-+

L  = LowBattery (1 bit)
C  = ConfigPending (1 bit)
P  = PiggybackAppAck (1 bit)
D  = DutyCycle (1 bit)
-  = reserved (1 bit)
B  = Booted (1 bit)
CF = ContentFormat (2 bits)
```

- **LowBattery**, **DutyCycle**, **Booted**: self-explanatory device state
  flags.
- **ConfigPending**: the device itself believes it has a configuration
  change pending.
- **PiggybackAppAck**: this Status frame itself also serves as the
  acknowledgement of an application-layer command sent to the device
  (e.g. a direct-execution setpoint change) — a device may fold the ACK
  for such a command into its next regular status push rather than
  sending a separate Answer frame for it. A host implementation must
  treat a Status frame carrying this flag as confirming the device's
  most recent outstanding command in addition to processing the status
  data itself.
- **ContentFormat** (2 bits) selects the layout of the status data entries
  that follow:
  - `0` — uncompressed: each entry is `[type][channel index][value bytes...]`.
  - `1` — same-channel: a single leading channel index, then repeated
    `[type][value bytes...]` entries all for that channel.
  - `2` — same-type: a single leading type, then repeated
    `[channel index][value bytes...]` entries all of that type.
- **Status data type/value-length table**: each `type` byte selects a data
  type (temperature, operating voltage, heating-controller state, etc.)
  whose value has a fixed, type-specific byte length. Decoding the
  concrete meaning of each channel's values is inherently
  device-type-specific and out of scope for this specification's core
  layer — treat status data as an opaque, typed sequence at the protocol
  level; per-device-type semantic decoding is a separate concern.

**Two independent, easily-confused RSSI values exist** and are a common
source of confusion:

- The Status frame's own RSSI byte (byte 1 above) reflects the signal
  strength *as measured by the reporting device itself* of whatever it
  last received. A raw byte value of `0x80` (`-128` once interpreted as
  signed) is a sentinel meaning "no reading available", not a real
  measurement — do not display it as a real RSSI figure.
- The signal strength of a packet *as measured by the access point's own
  coprocessor* is a completely different value, delivered out-of-band at
  the serial-protocol level: the leading byte of every Received event
  command (see `serial-protocol.md`), not anywhere inside the MAC/Status
  frame itself. This is what a real access-point's own user interface
  typically displays as "device RSSI" — not the Status frame's own field.
