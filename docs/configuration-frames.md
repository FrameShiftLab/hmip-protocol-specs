# Configuration frames: reading and writing device settings

Application FrameType 1 (Configuration, see `mac-application-frames.md`)
covers a whole family of sub-operations, distinguished by a request-type
byte immediately following the channel number. **Status: the frame
envelope and every specific byte layout below marked wire are verified on
the wire** (a real vendor access-point implementation observed performing
these exact operations against real devices, and against a from-scratch
reimplementation of the same operations independently confirmed to
interoperate). Which logical setting lives at which (list, index) address
is largely **derived from source analysis** of vendor server software and
device-description data, cross-checked against a handful of live reads
where noted.

## Shared frame shape

Every Configuration-family frame shares a 2-byte base:

```
[channel number][request type]
```

followed by request-type-specific fields. Application header:
`ResponseRequested = true`; `StayAwake = true` for the write sub-sequence
below when a write is pending, matching the sleeping-device discipline in
`native-device-links.md`.

## Request types

| Value | Name | Direction |
|---|---|---|
| 4 | Configuration Data Request | access point → device |
| 5 | Start Parameter Setting | access point → device |
| 6 | Commit Parameter Setting | access point → device |
| 8 | Set Parameter By Index | access point → device |
| 11 | Response Configuration Data | device → access point |
| 15 | Request Config Update | either |

Two additional request types (0x01 create-link, 0x02 remove-link,
0x03 request-link-partner-list, 0x0a response-link-partner-list, 0x0e
report-link-partner-problem) belong to the native-link mechanism — see
`native-device-links.md`.

## The "self" link-partner address

Several of the request types below carry a "link partner address" field —
this is the same field used for genuine device-to-device link operations
(see `native-device-links.md`), but reading or writing a device's *own*
configuration (as opposed to a link's per-partner parameters) uses a
reserved sentinel value for that field: address type "plain device
address" — a logical/internal classification, not a wire byte; every
address this spec has observed uses this one type, which has no marker
byte of its own on the wire (see `native-device-links.md`'s own note on
this same field) — value all-zero (3 zero bytes), and a link-partner
channel number of **literal `0`, unconditionally** — independent of
which channel on the target device is actually being configured. This is
a real, easily-miscoded detail: setting the link-partner channel number
equal to the *target* channel (a natural but incorrect assumption) has
been observed to plausibly cause the request to be silently rejected or
ignored by the device.

## Reading configuration: Configuration Data Request

```
[channel][0x04][link partner address: self-sentinel][link partner channel: 0][list number][start index]
```

The device answers with one or more Response Configuration Data frames
(request type 11), each carrying a chunk of `(index, value)` byte pairs —
**verified on the wire to be roughly 13–14 pairs per response**, not the
full requested list at once. To read the rest of a list, issue a further
Configuration Data Request with `start index` set to the highest index
value seen in the previous response (not simply "previous start + count
returned") — this matches the response's own indication of where the next
chunk should begin. The list is exhausted once the device signals
end-of-list in its response header (an out-of-range sentinel index).

**Response Configuration Data** frame payload — mirrors the same base
fields as the request that triggered it, rather than a shortened form:
`[channel][0x0b][link partner address: self-sentinel][link partner
channel][list number][end-of-list flag/last index][... (index, value)
byte pairs ...]` (8 header bytes before the pairs begin, not 4 — easy to
under-count if only glancing at the request's own, differently-shaped
base).

```
function decode_configuration_data_response(payload) -> ConfigurationDataResponse | null:
    if length(payload) < 2 or payload[1] != RESPONSE_CONFIGURATION_DATA (0x0b):
        return null
    if length(payload) < 8:
        return null   # truncated, not even the fixed header present
    channel                = payload[0]
    # payload[2:5] = link partner address (self-sentinel in practice), payload[5] = its channel - unused here
    list_number             = payload[6]
    last_parameter_id        = payload[7]
    end_of_list               = (last_parameter_id == END_OF_LIST_SENTINEL)
    pairs = []
    rest = payload[8:]
    for i in range(0, length(rest) - 1, 2):
        pairs.append((index = rest[i], value = rest[i + 1]))
    return ConfigurationDataResponse(channel, list_number, end_of_list, pairs)
```

A Configuration Data Request answered this way does **not** additionally
require, or receive, a separate Answer/ACK frame — the data-frame response
itself is the acknowledgement. A request sent with
`ApplicationResponseRequested = false` is answered identically; this is
useful for an access point implementation that wants to poll a value
without needing to track and re-send on a missing generic ACK.

## Writing configuration: three-frame sequence

Writing one or more configuration values to a device's given
(channel, list) address space is a strict 3-frame sequence, each frame
sent only after the device's Answer/ACK to the previous one (see
`serial-protocol.md`'s transmission discipline):

1. **Start Parameter Setting** —
   `[channel][0x05][link partner address: self-sentinel][link partner channel: 0][list number][link partner operation mode: 0]`
2. **Set Parameter By Index**, one or more of these, each carrying up to
   **9** `(index, value)` byte pairs (observed batch limit) —
   `[channel][0x08][index][value][index][value]...`
3. **Commit Parameter Setting** — `[channel][0x06]` (no further payload)

Each frame in the sequence needs its own, incrementing application
sequence number.

**Writing a non-default link parameter** (customizing a specific
device-to-device link's own configuration, as opposed to the device's own
general settings) uses this exact same 3-frame sequence, with the *real*
link-partner address and channel filled in instead of the self-sentinel.

## Locating a specific named setting

Which (channel, list, byte-index, bit-offset) address a given
human-meaningful setting (e.g. "low-battery threshold", "sample interval",
"show setpoint vs. room temperature on display") lives at is defined
per-device-type in the vendor's own device-description data, not
discoverable from the wire protocol alone. Each setting additionally has
a value-conversion rule (a linear scale/offset transform, a boolean bit,
a signed or enumerated integer, or a fixed-length string), and some
settings share a single byte via bit-packing (multiple sub-fields at
different bit offsets within the same index) — **a value at a
bit-packed index must be read, modified only at its own bits, and written
back, never blindly overwritten**, or neighboring settings sharing that
byte will be corrupted.

**Live-confirmed examples** (device-type-specific, given only as concrete
grounding, not as a general rule): a device's periodic sample interval
lives at list 1, index 147, as a plain integer scaled by ×10 (raw value
10 = 1.0 second); a climate-control device's "show setpoint instead of
room temperature on display" setting lives at list 1, index 12, bit 5.
Climate-related configuration in general has been observed to live on
list number **1**, not list 0 as might naively be assumed — a read/write
attempt against list 0 for climate settings will not find (or will not
correctly address) them.

**Worked example, verified on the wire** — the real 3-frame write
sequence for the sample-interval setting above (channel 1, target
device an `HMIP-SWDO`), Application header `0xc1` on every frame
(response requested + stay awake + FrameType=1, stay-awake set because
this is a multi-frame write to a device that needs to stay reachable
between frames):

```
frame 1 (Start Parameter Setting):  01 05 000000 00 01 00
frame 2 (Set Parameter By Index):   01 08 93 0a
frame 3 (Commit Parameter Setting): 01 06
```

- Frame 1: channel `01`, type `05`; self-sentinel link-partner address
  (`000000`) and channel (`00`); list number `01`; link-partner operation
  mode `00`.
- Frame 2: channel `01`, type `08`; one index/value pair — index `93`
  (147 decimal) = value `0a` (10 decimal, ×10-scaled = 1.0 second).
- Frame 3: channel `01`, type `06`, no further payload.

Each frame was sent only after the device's Answer to the previous one
(`02 <seq> 00`), matching the transmission discipline above; the Answer
to frame 2 specifically carried `StayAwake` set (`42 <seq> 00`) because
a further write (frame 3) was still pending for this device.

### Reading and writing a single bit-packed field

The one confirmed concrete case is a 1-bit boolean at a known bit offset
within an otherwise plain-integer byte (the "show setpoint" example
above, bit 5 of index 12) — the read-modify-write shape below generalizes
that to an arbitrary bit width in the obvious way (shift-and-mask), which
is a standard bit-manipulation technique rather than an HmIP-specific
claim; only the *specific* (list, index, bit) addresses and widths for
any given setting are protocol facts, and those must come from a real
device-description source or a live read, not from this generalization:

```
function extract_bits(byte_value, bit_offset, bit_width) -> int:
    return (byte_value >> bit_offset) & ((1 << bit_width) - 1)

function pack_bits(byte_value, bit_offset, bit_width, new_value) -> int:
    mask = ((1 << bit_width) - 1) << bit_offset
    return (byte_value & ~mask) | ((new_value << bit_offset) & mask)

# Writing ONE bit-packed setting without disturbing its neighbors:
function write_bit_packed_setting(device, channel, list_number, index, bit_offset, bit_width, new_value):
    current_byte = read_configuration_byte(device, channel, list_number, index)   # a full Configuration Data Request/response round trip
    updated_byte = pack_bits(current_byte, bit_offset, bit_width, new_value)
    write_configuration(device, channel, list_number, [(index, updated_byte)])     # the 3-frame sequence above
```

Skipping the read step (blindly writing a value computed from scratch,
assuming the rest of the byte is some known default) is exactly the
mistake the surrounding prose warns about — it silently corrupts whatever
other settings share that byte.

## Weekly heating program encoding

**Derived from source analysis; frame envelope and general write
mechanism verified on the wire (see above); the specific byte-packing
below verified against both a from-scratch encoder/decoder round-trip and
a real device's own reported programme after a live write.**

A thermostat-class device's weekly heating program lives in a
configuration list numbered `3 + program_slot_number` (i.e. its first
program slot is list 4, second is list 5, etc. — device types differ in
how many program slots they support). Each list encodes all seven days as
a single flat byte array:

- Days are ordered **Saturday, Sunday, Monday, Tuesday, Wednesday,
  Thursday, Friday** (not the more common Monday-first ordering).
- Each day has **13 time-slots**. Slot `k` (0-based, `k = day_index*13 +
  slot_index_within_day`) occupies 2 bytes starting at index `2k+1`
  (index 0 itself is unused/reserved):
  - Byte `2k+1`: bit 0 = bit 8 of the slot's end-time value; bits 1–6 =
    the slot's target temperature, encoded as `temperature × 2` (half-
    degree resolution, e.g. 21.5°C → raw value 43).
  - Byte `2k+2`: the low 8 bits of the end-time value.
  - **End time** is encoded in 5-minute units since midnight (i.e. divide
    minutes-since-midnight by 5 to encode, multiply by 5 to decode);
    unused trailing slots in a day are conventionally filled with
    end-time = 1440 minutes (24:00) and a repeated/default temperature.
- A full week is therefore `7 days × 13 slots × 2 bytes = 182 bytes`,
  written as: Start Parameter Setting (targeting the program's list
  number) → up to 21 Set Parameter By Index frames of ≤9 pairs each →
  Commit Parameter Setting — i.e. exactly the generic write sequence
  above, with no special-casing beyond the list number and byte layout.
- **Reading a program back** uses the ordinary Configuration Data Request
  mechanism above (list = `3 + program_slot_number`), decoded by exactly
  reversing this same byte layout. A live read of all 13–14-pair chunks
  covering a full 182-byte program has not yet been independently
  confirmed end-to-end (only a partial, first-two-chunks read has been),
  though the chunking mechanism itself (see "Reading configuration"
  above) is confirmed general-purpose.

### Encoding and decoding a week program

```
WEEK_DAYS = [Saturday, Sunday, Monday, Tuesday, Wednesday, Thursday, Friday]   # wire order
SLOTS_PER_DAY = 13
END_OF_DAY_MINUTES = 1440

# `schedule[day]` = a list of up to 13 (end_time_minutes, temperature_celsius)
# slots for that day, already validated (multiples of 5 minutes, strictly
# increasing, last one ending at 1440) and padded with trailing
# (1440, previous_temperature) slots up to exactly 13.

function encode_week_program(schedule) -> list of (index, value):
    pairs = []
    for day_number, day in enumerate(WEEK_DAYS):
        slots = pad_to_13_slots(schedule[day])   # see validation rules above
        for slot_number, (end_minutes, temperature) in enumerate(slots):
            k = day_number * SLOTS_PER_DAY + slot_number
            end_units = end_minutes / 5                              # 5-minute units
            temperature_raw = round(temperature * 2)                  # half-degree steps
            byte_a = (end_units >> 8 & 1) | ((temperature_raw & 0x3F) << 1)
            byte_b = end_units & 0xFF
            pairs.append((index = 2*k + 1, value = byte_a))
            pairs.append((index = 2*k + 2, value = byte_b))
    return pairs   # write via Start Parameter Setting -> Set Parameter By Index (<=9 pairs/frame) -> Commit

function decode_week_program(index_value_pairs) -> schedule:
    # index_value_pairs: the full 182 bytes' worth of (index, value), from
    # one or more paged Configuration Data Request/Response round trips
    by_index = map(index_value_pairs)   # index -> value, for random access
    schedule = {}
    for day_number, day in enumerate(WEEK_DAYS):
        slots = []
        for slot_number in range(0, SLOTS_PER_DAY):
            k = day_number * SLOTS_PER_DAY + slot_number
            byte_a = by_index[2*k + 1]
            byte_b = by_index[2*k + 2]
            end_units = ((byte_a & 1) << 8) | byte_b
            temperature_raw = (byte_a >> 1) & 0x3F
            slots.append((end_minutes = end_units * 5, temperature = temperature_raw / 2.0))
        schedule[day] = slots
    return schedule
```

### Worked example, verified on the wire — a real week-program write

A real write of two time-slots to `HmIP-WTH-2`'s Monday, profile 1
(`P1_ENDTIME_MONDAY_1=360`, `P1_TEMPERATURE_MONDAY_1=17.0`,
`P1_ENDTIME_MONDAY_2=1440`, `P1_TEMPERATURE_MONDAY_2=21.0`), Application
header `0xc1` on the Start/Commit frames (response requested + stay
awake) and `0x81` on the Set-By-Index frame in between (stay-awake
clear — this device is a burst listener, not an event listener, see
`native-device-links.md`):

```
frame 1 (Start Parameter Setting):  01 05 000000 00 04 00
frame 2 (Set Parameter By Index):   01 08 37 55 38 20
frame 3 (Commit Parameter Setting): 01 06
```

- Frame 1: list number `04` — matches `3 + program_slot_number` for
  profile 1 (`3 + 1 = 4`) exactly, confirming the list-number formula
  above against a real profile-1 write.
- Frame 2 carries only **one** of the two requested slots' worth of
  bytes (index/value pairs `37`/`55` and `38`/`20`), not four bytes for
  both slots as the request specified. Decoding those two bytes against
  the encoding formula above: slot `k=27` (Monday, slot index 1 — the
  *second* time-slot) → `end_units = 0x38 & 0xFF | ((0x37 & 1) << 8) =
  0x120 = 288` → `288 × 5 = 1440` minutes, and `temperature_raw =
  (0x37 >> 1) & 0x3F = 0x2A = 42` → `42 / 2.0 = 21.0`°C — both match the
  requested values for slot 2 exactly, confirming the byte-packing
  formula byte-for-byte against a real device. Slot 1's own bytes
  (`P1_ENDTIME_MONDAY_1`/`P1_TEMPERATURE_MONDAY_1`) were never
  transmitted at all — both of that slot's requested values (`360`
  minutes, `17.0`°C) matched this parameter's own declared **default**
  value (confirmed from the same live paramset schema query used to
  build this write), so real ReGa-side software appears to diff a write
  against its own last-known state and only transmits bytes that
  actually change, rather than always sending every requested
  (index, value) pair. This is a real, observed optimization, not
  something this spec's own write algorithm needs to replicate for
  correctness — sending the unchanged pair too is harmless, just an
  extra 2 bytes.
- Each frame was answered with an ordinary Answer/ACK before the next
  was sent, matching the transmission discipline documented above.

## Direct Execution Command frames (Application FrameType 6)

**Verified on the wire** — real captures of an access point issuing a
live setpoint change, a boost-mode toggle, and a vacation/party-mode
write to a real `HmIP-WTH-2`. Unlike the Configuration frames above (a
device's stored *settings*), a Direct Execution Command is a live,
immediate value change — sent as a single frame (`ApplicationHeader`
FrameType 6), no Start/Commit sequence, and not written to the device's
persistent configuration list.

Every real example captured shares the same 4-byte-minimum shape:

```
[0x80][0x01][command][value...]
```

- The leading `0x80` and `0x01` bytes were constant across every command
  variant observed (setpoint, boost, party mode) — their exact meaning
  isn't confirmed; `0x80` may encode a channel number with a flag bit set
  (every observation here happened to target channel 0, so the
  channel-encoding scheme itself can't be distinguished from a constant
  flag byte with only one channel value on record).
- `[command]` is a one-byte discriminator selecting which live value is
  being set:

| Value | Command |
|---|---|
| `0x00` | Boost mode |
| `0x02` | Set-point temperature |
| `0x03` | Party (vacation) mode |

**Set-point temperature** (`command = 0x02`) — real example, setting
`19.5`°C:

```
80 01 02 27
```

`0x27` = 39 decimal = `19.5 × 2` — the same half-degree-step encoding
used throughout this protocol (see the weekly-program encoding below).
This one field's value directly cross-checks against the RPC call that
produced it (a `setValue` of exactly `19.5`), so the temperature-scaling
claim here is high-confidence; the leading `80 01 02` prefix is
confirmed only for this specific channel/command combination.

**Boost mode** (`command = 0x00`) — real examples, enabling then
disabling:

```
enable:  80 01 00 30
disable: 80 01 00 20
```

The two payloads differ only in bit 4 of the trailing byte (`0x30` vs
`0x20`) — consistent with a bitfield rather than a plain boolean 0/1, but
the meaning of the other set bits (`0x20`, present in both) isn't
confirmed; a boolean flag on a shared bitfield byte is a plausible
reading, not a verified one.

**Party/vacation mode** (`command = 0x03`) — **verified on the wire, encoding
fully decoded**: six real examples, each with deliberately different
start/end dates and times — including two isolating a year change with
every other field held fixed, and one deliberately crossing a
day/month/year boundary within a single start/end pair — cross-referenced
against each other to recover the field layout with confidence (a single
example genuinely wasn't enough — see below for why six was):

| # | `PARTY_TIME_START` | `PARTY_TIME_END` | 7 trailing bytes |
|---|---|---|---|
| 1 | `2026_09_29 10:00` | `2026_09_30 10:00` | `3c 1d 3c 1e 99 1a 1a` |
| 2 | `2026_10_15 10:00` | `2026_10_16 10:00` | `3c 0f 3c 10 aa 1a 1a` |
| 3 | `2027_03_05 23:45` | `2027_03_06 08:15` | `8e 85 31 86 33 1b 1b` |
| 4 | `2026_06_15 12:00` | `2026_06_16 12:00` | `48 0f 48 10 66 1a 1a` |
| 5 | `2029_06_15 12:00` | `2029_06_16 12:00` | `48 0f 48 10 66 1d 1d` |
| 6 | `2026_12_31 23:50` | `2027_01_01 00:10` | `8f 1f 01 01 c1 1a 1b` |

Examples 4 and 5 are identical in every field except the year (2026 vs.
2029, a 3-year jump) — a clean isolation showing byte-for-byte which
bytes the year actually changes. Example 6 crosses a day, month, *and*
year boundary within one start/end pair (New Year's Eve → New Year's
Day), the only way to test whether `PARTY_TIME_START` and
`PARTY_TIME_END` are genuinely independent 7-byte fields.

Full frame, example 3: `80 01 03 8e 85 31 86 33 1b 1b`.

The 7 bytes decode as:

```
[start_10min][start_day][end_10min][end_day][month_pair][start_yr][end_yr]
```

- **`start_10min`** (byte 0) — minutes since midnight of the start time,
  divided by 10 and floored: `floor(minute_of_day / 10)`. Example 3's
  `23:45` = minute 1425 → `floor(1425/10) = 142 = 0x8e`, matching
  exactly. Example 6's `23:50` = minute 1430 → `floor(1430/10) = 143 =
  0x8f`, also matching, and confirms the field is a plain 0–143 value
  with no separate high-bit flag (143 is simply the largest value this
  field can take — `floor(1439/10)`, one minute before midnight — and
  its binary representation naturally sets bit 7, which is not a
  meaningful flag, just arithmetic). This also reveals a real
  device-side behavior: party-mode start/end times are only settable to
  **10-minute granularity** — the `:45` and `:15` in example 3's request
  got silently floored to `:40` and `:10` on the wire (`23:45`→1420,
  `08:15`→490; `floor(490/10)=49=0x31`, matching byte 2 below).
- **`start_day`** (byte 1) — the day-of-month of the start date, in the
  low 7 bits (`byte & 0x7f`): `0x1d & 0x7f = 29` (Sep **29**), `0x0f &
  0x7f = 15` (Oct **15**), `0x85 & 0x7f = 5` (Mar **5**), `0x0f & 0x7f =
  15` (Jun **15**, twice, examples 4–5), `0x1f & 0x7f = 31` (Dec **31**,
  example 6) — exact matches in all six. Bit 7 was set only in example 3
  and nowhere else, including examples 4 and 5 (which isolate a 3-year
  jump, 2026→2029, with every other field — including this day value —
  held identical) and example 6's `end_day` (byte 3, see below), which
  is 1 January **2027** — the same year as example 3 — yet has bit 7
  clear. That directly refutes bit 7 as any kind of year-related flag:
  if it were, example 6's end-of-pair date (2027) would set it just as
  example 3's date (also 2027) supposedly did, and it doesn't. With five
  further independent, deliberately-varied examples never reproducing
  it, example 3's bit 7 is now treated as a one-off anomaly in that
  specific capture — not part of this field's real encoding — though
  its actual cause remains unexplained.
- **`end_10min`** (byte 2) — same `floor(minute_of_day/10)` encoding as
  byte 0, for the end time. Example 3: `08:15` = minute 495 →
  `floor(495/10) = 49 = 0x31`, matching. Example 6: `00:10` = minute 10
  → `floor(10/10) = 1 = 0x01`, matching.
- **`end_day`** (byte 3) — same day-of-month encoding as byte 1 (see
  above for why the bit-7 theory is refuted using this exact byte), for
  the end date: `0x1e & 0x7f = 30`, `0x10 & 0x7f = 16`, `0x86 & 0x7f =
  6`, `0x10 & 0x7f = 16` (examples 4–5), `0x01 & 0x7f = 1` (Jan **1**,
  example 6) — all exact matches, bit 7 clear in every case.
- **`month_pair`** (byte 4) — start month and end month, packed one per
  nibble, high nibble first: `(start_month << 4) | end_month`. Example
  1: September→September = `(9<<4)|9 = 0x99`. Example 2:
  October→October = `(10<<4)|10 = 0xaa`. Example 3: March→March =
  `(3<<4)|3 = 0x33`. Examples 4–5: June→June = `(6<<4)|6 = 0x66`.
  Example 6, the month-boundary-crossing case: December→January =
  `(12<<4)|1 = 0xc1` — matching exactly, confirming both the nibble
  order (`high=start, low=end`) and that a month-crossing pair encodes
  correctly with no special-casing.
- **`start_yr` / `end_yr`** (bytes 5, 6) — year minus 2000, one byte
  each: `0x1a = 26` (2026, examples 1–2 and both bytes of 4), `0x1b = 27`
  (2027, both bytes of example 3), `0x1d = 29` (2029, both bytes of
  example 5) — exact matches throughout, and confirmed as a dedicated,
  independent field (not related to the day byte's bit 7, see above).
  Example 6, the year-boundary-crossing case, is the decisive test:
  `start_yr = 0x1a = 26` (2026) and `end_yr = 0x1b = 27` (2027) —
  different values in the same frame, exactly matching the real
  December 2026 → January 2027 transition requested. This confirms
  `start_yr` and `end_yr` are genuinely independent bytes that can and
  do differ from each other.

Every field above round-trips exactly against all six real, independently
varied inputs — including the two purpose-built isolation tests (a
year-only change, and a day/month/year boundary crossing) — and this is
now a fully confirmed encoding. The only open question left is what
example 3's stray, unreproduced bit-7 actually was; it's confirmed *not*
to be part of this field's normal encoding, but its real cause (if any)
is unknown.

Each Direct Execution Command frame was answered with an ordinary
Answer/ACK (`FrameType 2`), same as any other Application frame — no
special acknowledgement shape observed.

**Not confirmed**: the byte layout for mode (auto/manual), window-state,
active-profile, or valve-adaption Direct Execution Commands — no traffic
performing those specific commands was captured tonight.

## Not covered here

Over-the-air firmware update (OTAU), remote live update, and OEM
container frame families exist (see the frame-type table in
`mac-application-frames.md`) but have not been investigated for this
specification.
