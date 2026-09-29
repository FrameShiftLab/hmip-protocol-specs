# Serial (UART) protocol between host and coprocessor

**Status: verified on the wire**, unless a section says otherwise. This
layer carries every HmIP MAC frame between a host implementation and the
radio coprocessor module, plus a small set of coprocessor-management
commands.

## Frame envelope

```
[0xFD] [Length: u16, big-endian] [Destination] [Sequence counter] [Command] [Data...] [CRC: u16, big-endian]
```

- **Sync byte**: `0xFD`, always the first byte of a frame.
- **Length**: 2 bytes, big-endian, counts the bytes from `Destination`
  through the end of `Data` inclusive (does not include the sync byte,
  the length field itself, or the trailing CRC).
- **Destination**: a subsystem/address byte. Two values matter for HmIP:
  - `0x02` — HmIP subsystem. The module reports HmIP RF events and accepts
    HmIP commands on this destination.
  - `0x01` — a second, unrelated subsystem (classic sub-1 GHz "BidCoS-RF"
    protocol family) multiplexed on the same physical link via this same
    byte. A host that only cares about HmIP simply ignores frames whose
    Destination isn't `0x02`.
- **Sequence counter**: 1 byte, incremented by the sender per frame. Used
  to correlate a Response (see below) with the command it answers.
- **Command**: 1 byte, see the command tables below.
- **Data**: command-specific payload, length implied by the frame's
  `Length` field.
- **CRC**: 2 bytes, big-endian. CRC-16, initial value `0xFFFF`, polynomial
  `0x8005`, MSB-first, computed over `Length`..`Data` inclusive (the length
  field itself is covered; only the sync byte and the CRC field itself are
  excluded from the checksum).

There is exactly one such envelope shared by every command family on this
link — an HmIP command and a legacy coprocessor-management command differ
only in which `Command` byte space they use, not in framing.

**Byte stuffing, verified on the wire — easy to miss, breaks framing if
skipped.** After the sync byte, every occurrence of the sync byte's own
value (`0xFD`) *or* of a separate escape byte value (`0xFC`) anywhere in
the rest of the frame (length field through CRC, inclusive — i.e.
everything the `Length` field and the CRC computation both cover) is
escaped: replace it with the two-byte sequence `0xFC` followed by that
original byte. A reader must undo this substitution (drop each `0xFC`
byte and keep the byte that follows it, verbatim) *before* interpreting
the length field, computing the CRC, or doing anything else with the
unescaped bytes — `Length` and the CRC are both computed over the
unescaped form. This has nothing to do with the sync byte search itself:
only one real `0xFD` exists per frame (the leading sync byte); any other
`0xFD` byte value that legitimately occurs inside the frame body is
always escaped.

### Reading a frame from a byte stream

```
function read_next_unescaped_byte(byte_source) -> u8:
    b = byte_source.next()
    if b == ESCAPE_BYTE (0xFC):
        b = byte_source.next()   # the real byte follows the escape marker
    return b

function read_frame(byte_source) -> Frame:
    # 1. Find the sync byte. Bytes before it (if any) are noise/resync.
    #    The sync byte itself is never escaped and is not part of the
    #    unescaped byte stream read below.
    while byte_source.next() != SYNC_BYTE (0xFD):
        continue

    # 2. Read exactly 2 unescaped bytes: the length field. This tells us
    #    how many more unescaped bytes to expect.
    length_bytes = [read_next_unescaped_byte(byte_source), read_next_unescaped_byte(byte_source)]
    declared_length = (length_bytes[0] << 8) | length_bytes[1]

    # 3. Now read exactly that many more unescaped bytes: dest+seq+command+data
    #    (declared_length bytes) followed by the 2-byte CRC.
    rest = []
    for i in range(0, declared_length + 2):
        rest.append(read_next_unescaped_byte(byte_source))

    unescaped = length_bytes + rest

    # 4. Split the unescaped bytes into fields.
    declared_length      = (unescaped[0] << 8) | unescaped[1]
    destination           = unescaped[2]
    sequence_counter       = unescaped[3]
    command                = unescaped[4]
    data_length             = declared_length - 3   # length counts dest+seq+command+data
    data                     = unescaped[5 : 5 + data_length]
    received_crc            = (unescaped[5+data_length] << 8) | unescaped[5+data_length+1]

    # 5. Validate.
    computed_crc = crc16(unescaped[0 : 5 + data_length])   # length field through data
    if received_crc != computed_crc:
        raise FrameError("CRC mismatch")

    return Frame(destination, sequence_counter, command, data)
```

### CRC-16 computation

Bit-by-bit, matching the `0x8005`/MSB-first/`0xFFFF`-init parameters
stated above (a standard CRC-16 variant, not HmIP-specific, but spelled
out here since implementations vary in exactly how they apply the
polynomial):

```
function crc16(bytes) -> u16:
    crc = 0xFFFF
    for byte in bytes:
        for bit_index in 0..7:              # most significant bit of `byte` first
            bit = (byte >> (7 - bit_index)) & 1
            msb_of_crc = (crc >> 15) & 1
            crc = (crc << 1) & 0xFFFF
            if msb_of_crc xor bit:
                crc = crc xor 0x8005
    return crc
```

## HmIP command set

Command byte values a host sends to, or receives from, the coprocessor on
the HmIP destination:

| Value | Name | Direction | Data |
|---|---|---|---|
| 0 | Set default network address | host→module | — |
| 1 | Get default network address | host→module | — |
| 2 | Set network key | host→module | — |
| **3** | **Send protocol frame** | host→module | `[burst mode: u8][MAC frame bytes...]` — transmit a MAC frame (see `mac-application-frames.md`) with the given retransmission strategy |
| **4** | **Add link partner** | host→module | `[device address: 3 bytes][operation mode: u8][router address: 3 bytes]` — register a peer in the module's own link/routing table |
| **5** | **Remove link partner** | host→module | `[device address: 3 bytes]` — **verified on the wire**, real captures observed 15 separate invocations, each carrying only the 3-byte address, not the full `(address, mode, router)` tuple command 4 takes |
| **6** | **Response** | module→host | `[status code: u8][data...]`, correlated to the triggering command by sequence counter — see "Response codes" below |
| **7** | **Received event** | module→host | `[RSSI: signed u8][MAC frame bytes...]` — the module received a frame over RF; the leading byte is the module's own per-packet hardware signal-strength measurement (negate the signed byte to get a dBm-like figure), immediately followed by the raw MAC frame bytes |
| 8 | Set security counter | host→module | — |
| 9 | Store security counter | host→module | — |
| 10 | Get security counter | host→module | — |
| 11–14 | CSMA/send-attempt defaults get/set | host→module | — |
| 15 | Add listener address | host→module | — |
| 16 | Remove listener address | host→module | — |
| **17** | **Get inclusion data** | host→module | `[SGTIN: 12 bytes][nonce: 4 bytes][one-time key: 8 bytes]` of the joining device — requests the module's own hardware-computed key-exchange material for an in-progress device inclusion (see `security-and-inclusion.md` for a full worked example). The response arrives via the generic Response command (6): `[status][36-byte key material]`, **verified on the wire** |
| **18** | **Get link partner list** | host→module | empty request; the Response (command 6) payload is `[status][device address: 3 bytes]...`, repeated — a flat list of addresses, no per-entry channel byte (unlike the Application-layer link-partner-list mechanism in `native-device-links.md`) — **verified on the wire**, a real 15-entry response observed |
| 19–20 | Get/set encrypted network key | host→module | — ; command 20 (the "set" variant) is **observed on the wire** being sent with a real, large (66-byte in one capture) payload between "Get inclusion data" and the Inclusion Accept frame during a real inclusion — its own field layout isn't decoded in this spec, see `security-and-inclusion.md` |
| 21–23 | OTAU interval send/response/cancel | both | over-the-air-update related; not otherwise documented here |
| 24 | Send inclusion accept | host→module | a structured variant of sending an inclusion-accept frame with individual fields, used only by a device-local-key inclusion path — **not** used for the standard master-key/cloud-keyserver inclusion path, which instead sends the accept frame as an ordinary **Send protocol frame** command (verified — see `security-and-inclusion.md`) |
| 25–30 | adapter MIC / connection data / inclusion-accept-data variants | host→module | not otherwise documented here |

**Burst mode** (the 1-byte prefix on the Send protocol frame command) selects how many
times, and in what pattern, the module retransmits the frame over RF. The
correct burst mode for a given transmission is derived from the target
device's own declared listener mode (a value carried in its
device-inclusion request, see `security-and-inclusion.md`) — the module
retransmits the same frame several times, spaced out, so a device that
isn't listening continuously still has a good chance of catching one of
the repeats. **Sending a Normal-mode (single, non-repeated) transmission
to a device whose declared listener mode calls for burst delivery has a
real chance of the frame simply never arriving** — this was directly
observed causing silent, indefinite retry loops.

**Real observed listener mode values** (the low nibble of the packed
device role/mode flags field in a device's own Inclusion Request, see
`security-and-inclusion.md`):

| Value | Name | Burst mode selected | Real example |
|---|---|---|---|
| 1 | Single burst listener | Burst | — |
| 3 | Triple burst listener | Triple burst | `HmIP-WTH-2` (wall thermostat) |
| 9 | Cyclic + single burst listener | Burst | — |
| 11 | Cyclic + triple burst listener | Triple burst | `HmIP-eTRV-2` (radiator thermostat) |
| anything else | (not cyclic/burst) | Normal (single transmission) | — |

The "Cyclic + ..." values mean the device's own receiver genuinely
cycles on and off on a short period, only listening in brief windows —
for these, the module's repeated burst transmission is what reliably
catches one of those windows, not any explicit wait-for-the-device
mechanism on the host side. The plain (non-"Cyclic +") burst values
still call for the same repeated-transmission burst mode (presumably for
RF reliability rather than a narrow listen window, though this spec
doesn't independently confirm that distinction at the hardware level) —
either way, a host implementation only needs to pick the matching burst
mode value above; no separate synchronization step is needed for *any*
of these listener modes. That's different from the true "event listener"
class (no periodic listening at all, reachable only right after the
device's own transmission — typically battery-powered sensors), which
*does* need the wait-for-transmission mechanism described in
`native-device-links.md`'s "Sleeping devices" section — see that
document for the distinction and why conflating the two is a real,
observed mistake.

## Response codes

A Response frame's first data byte is a status code:

| Code | Meaning |
|---|---|
| 1, 9 | OK |
| 2 | **Busy** — the module is currently transmitting another frame and refuses this command; see "Transmission discipline" below |
| 3 | Timeout |
| 4 | No reply (from the RF peer) |
| 5 | Routed |
| 6 | Duty cycle limit |
| 8 | Error |
| 10 | Wrong input |

## Transmission discipline (important, non-obvious finding)

**The coprocessor module can only have one RF transmission in flight at a
time.** Issuing a second Send protocol frame command (or any other command that
triggers a transmission) while a previous one is still on the air is
refused immediately with response code 2 (Busy), and **the frame is
silently dropped** — it is not queued.

This matters a great deal in practice: a burst-mode transmission to a
thermostat-class device can keep the module busy for on the order of
several seconds. If a host implementation queues multiple outgoing frames
without waiting for each one's Response before sending the next, frames
addressed to *other* devices — including time-critical acknowledgement
frames a device is actively waiting to receive — can be silently lost
during that busy window. Observed real-world symptom: devices' own status
LEDs indicating "link/command not confirmed" (see `native-device-links.md`)
purely because an unrelated, concurrent transmission was still occupying
the module when the relevant frame was sent.

**Correct discipline, matching how vendor server software behaves:** send
one command that triggers a transmission, wait for its Response
(correlated by sequence counter) before sending the next one to the module,
regardless of which device each is addressed to. On a Busy response, retry
the same command with a fresh sequence counter after a short delay rather
than abandoning it.

This is a genuinely valuable, non-obvious finding: it is easy to build a
host implementation that appears to work most of the time — most individual
transmissions succeed — while intermittently losing frames under load in a
way that manifests only as device-side "not confirmed" indicators, with no
error surfaced anywhere on the host side.

### Transmission discipline as an algorithm

The single most valuable "how do I not get this wrong" piece of this
specification. `outgoing_queue` here is per-implementation — one shared
queue for all devices, not one per device, since the constraint (one RF
transmission in flight, period) is per *module*, not per device:

```
# state, shared across the whole implementation:
outgoing_queue = []            # commands waiting to be sent
sequence_counter = 0            # wraps at 256
pending_response = None         # the one command currently awaiting a Response

function enqueue(command, data):
    outgoing_queue.append((command, data))
    try_send_next()

function try_send_next():
    if pending_response is not None:
        return   # a transmission is already in flight - wait for its Response
    if outgoing_queue is empty:
        return
    (command, data) = outgoing_queue.pop_front()
    counter = sequence_counter
    sequence_counter = (sequence_counter + 1) mod 256
    pending_response = {
        command: command, data: data, counter: counter,
        attempt: 1, sent_at: now(),
    }
    send_frame(destination=HMIP, sequence_counter=counter, command=command, data=data)

# called whenever a Response frame (command byte 6) arrives:
function on_response(sequence_counter, status_code):
    if pending_response is None or sequence_counter != pending_response.counter:
        return   # stray/late response, not for the currently in-flight command
    if status_code == BUSY (2):
        if pending_response.attempt >= MAX_ATTEMPTS:
            pending_response = None
            report_failure(pending_response.command, "gave up after repeated Busy")
            try_send_next()   # move on to the next queued command
            return
        wait(RETRY_DELAY)   # a short, fixed or lightly-backed-off delay
        counter = sequence_counter
        sequence_counter = (sequence_counter + 1) mod 256
        pending_response.counter = counter
        pending_response.attempt += 1
        pending_response.sent_at = now()   # each retry is a fresh transmission - the watchdog below must time out against the most recent attempt, not the original one
        send_frame(destination=HMIP, sequence_counter=counter,
                   command=pending_response.command, data=pending_response.data)
        return   # still in flight, same command - do not advance the queue
    # any non-Busy status (OK, Timeout, No-Reply, ...) frees the module
    # for the next command, whether or not this one ultimately succeeded
    pending_response = None
    try_send_next()

# a watchdog, in case a Response is somehow never delivered at all:
function on_tick():
    if pending_response is not None and now() - pending_response.sent_at > RESPONSE_TIMEOUT:
        failed_command = pending_response.command   # capture before clearing
        pending_response = None
        report_failure(failed_command, "no Response received")
        try_send_next()
```

The essential property this preserves: **`pending_response` is a single
slot, not a set** — nothing is ever sent to the module while it is
non-empty, regardless of which device the next queued command targets.
