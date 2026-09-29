# Worked walkthrough: a device inclusion, end to end

This document is a synthesis, not new reference material: it walks
through one complete scenario — a device joining an access point,
through to its first piece of ordinary post-inclusion traffic — in
chronological order, citing the exact fields and (where a real capture
exists) the exact bytes already established elsewhere in this spec. For
full field-by-field reference, follow the links out to each section
rather than expecting the derivation repeated here.

**Read this after
[`serial-protocol.md`](serial-protocol.md),
[`mac-application-frames.md`](mac-application-frames.md), and
[`security-and-inclusion.md`](security-and-inclusion.md)** — this page
assumes their terminology and doesn't redefine it (see
[`glossary.md`](glossary.md) for quick lookups). The overall message
flow below is the same one already diagrammed in
[`security-and-inclusion.md`'s sequence diagram](security-and-inclusion.md#device-inclusion-handshake);
this page narrates it linearly, with bytes where bytes exist.

## A note on grounding, up front

**Steps 1, 2, 4, and 5 below all reuse real captured bytes** — two
separate real capture sessions cover, between them, an entire inclusion
handshake (Steps 1–4, one device: a real `HMIP-SWDO`) and ordinary
post-inclusion traffic (Step 5, a different device and access point). No
concrete captured hex exists in this repository for the cloud
key-exchange call itself (Step 3) — that step is walked through
structurally. Each step says explicitly which kind of grounding it has.

## Step 1 — Device sends an Inclusion Request frame

*Verified on the wire — real captured bytes.*

A joining device periodically transmits an Inclusion Request frame
(NetworkManagement `ContentType`, MAC-layer security **disabled** since
no network key is shared yet) — observed to repeat roughly every 10
seconds until accepted. Its 39-byte payload carries, among other fields,
a 12-byte [SGTIN](glossary.md#sgtin), a 2-byte manufacturer code, a
4-byte manufacturer device type, the device's declared
[listener mode](glossary.md#listener-mode), a 4-byte nonce, a 4-byte MIC,
and an 8-byte one-time-key contribution. Full field table and a real
worked byte example:
[`security-and-inclusion.md` § Inclusion Request frame](security-and-inclusion.md#inclusion-request-frame-device--access-point).
That real example's manufacturer code/device type pair (1, 258) is
exactly what [`device-catalog.md`](device-catalog.md) maps to
`HMIP-SWDO` — independently cross-checked between two unrelated real
captures, and the same device this walkthrough's Steps 2 and 4 follow
through the rest of its own inclusion. That lookup isn't needed for the
handshake itself to proceed, only for a host implementation that wants
to show a human-readable model.

## Step 2 — Access point requests its own key material

*Verified on the wire — real captured request and response bytes.*

The access point issues serial command **17 (Get inclusion data)** — see
[`serial-protocol.md` § HmIP command set](serial-protocol.md#hmip-command-set)
— to its own coprocessor module. The coprocessor computes a block of
key-exchange material using its own secret, per-module hardware key
(never exposed to the host), combined with the request's SGTIN and
nonce, plus a freshly generated random one-time-key contribution and
nonce on the access-point side. This is **not** a passthrough of
anything the device sent. Full detail, a real worked request/response
example, and the
[CCM*](glossary.md#ccm)-style [CBC-MAC](glossary.md#cbc-mac)/keystream
construction involved:
[`security-and-inclusion.md` § Access-point-side key material](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

## Step 3 — Cloud key exchange (cloud-mode inclusion only)

*Verified on the wire (per `security-and-inclusion.md`'s own marking) —
the request/response **shape** was used against the real production
service and accepted; no concrete example JSON with real hex field
values is given anywhere in this spec, so none is reproduced here.*

If the device's per-device key wasn't pre-provisioned out of band, the
access point instead resolves the key exchange via
`HTTPS POST https://secgtw.homematic.com:8443/ccm/gateway`, sending
`accessPointNonce`, `networkKeyEncrypted`, and `networkKeyMIC` — exactly
the values Step 2 would otherwise have produced entirely locally — and
receiving back `jdEncNWK` / `jdEncNWKMIC` / `nonceKS`, which map directly
onto the Inclusion Accept frame's own fields in Step 4. Full request/
response shape and the specific failure mode this project hit and
resolved (a placeholder key producing a silent `notAnswerable`
rejection):
[`security-and-inclusion.md` § Cloud key exchange](security-and-inclusion.md#cloud-key-exchange).

Skip straight to Step 4 for a local-only inclusion (see
[`security-and-inclusion.md` § Local-only key exchange mode](security-and-inclusion.md#local-only-key-exchange-mode)
— derived from source analysis, not independently confirmed end-to-end).

## Step 4 — Access point registers the device and sends the accept

*Verified on the wire — real captured bytes for both steps below, from
the same real inclusion as Steps 1 and 2.*

Two things happen, in order:

1. **Add link partner** (serial command 4) registers the new device
   address in the coprocessor's own routing table — sent once,
   immediately before the accept frame. See
   [`serial-protocol.md` § HmIP command set](serial-protocol.md#hmip-command-set)
   and the real worked example in
   [`security-and-inclusion.md` § Inclusion Accept frame](security-and-inclusion.md#inclusion-accept-frame-access-point--device).
2. The 44-byte **Inclusion Accept frame** (used inclusion mode, NonceKS,
   one-time-key MIC, one-time key, encrypted network key, new device
   address, new MAC sequence number, network key MIC — full table and a
   real worked example:
   [`security-and-inclusion.md` § Inclusion Accept frame](security-and-inclusion.md#inclusion-accept-frame-access-point--device))
   is sent as an ordinary serial command 3 (**Send protocol frame**),
   **prefixed with the burst mode matching the joining device's own
   declared listener mode from Step 1**. Getting this burst mode wrong
   is the single most consequential mistake in this whole handshake —
   the source file documents it as a directly-observed root cause of
   inclusions that looked correct in every other respect (crypto, frame
   layout, prior link-partner registration) but never completed, with no
   error surfaced on the access-point side. (The real capture behind this
   walkthrough used burst mode `0x00`, a single transmission — evidently
   correct for this particular device's own declared listener mode.)

## Step 5 — Device accepts; ordinary traffic begins

*Verified on the wire — real captured bytes, from the second capture
session noted above.*

Once the device accepts, it stops re-sending its Inclusion Request and
starts sending ordinary Application-layer frames — status updates, time
requests, and so on — secured with the network key just established.
This is exactly the kind of traffic
[`mac-application-frames.md`'s worked example](mac-application-frames.md#worked-example-a-real-captured-status-frame)
captures — **from a separate real capture session** than Steps 1–4
above (a different device and access point; Steps 1–4's own capture
doesn't continue far enough past acceptance to show this device's first
post-inclusion frame). Reusing that example here, byte-for-byte, in the
hex style established there (bare, lowercase, space-separated per byte,
multi-byte fields run together — see
[`index.md`'s conventions section](index.md#conventions-used-in-this-specification)):

```
10 01 8e  2cf049  b1235a  00000067 1052057c  85 02  04 80 01 01 00 de 43 03 00 1a 04 01 68 00 00 0d 01 00
```

Walking it at a glance (full byte-by-byte breakdown, including why each
field has the value it does, lives at the link above — not repeated
here):

- `10` — MAC header byte 0: `ContentType`=Application, `SecurityControl`=ENABLED (1), `Cyclic`/`Fragmentation` clear. The device is already flagging this frame as secured, using the network key Step 4 just delivered.
- `01 8e` — MAC header bytes 1–2: no IP address extension, `PiggybackAck`=1; `HomematicIPVersion`=8, `LocalHopLimit`=7.
- `2cf049` — MAC source address: this (different, separately-captured) device's own new address, assigned the same way Step 4's "new device address" field assigns one.
- `b1235a` — MAC destination address: the access point.
- `00000067 1052057c` — SecurityNumber + IntegrityCode: real, non-zero values here (`00000067` / `1052057c`). This is *not* a case of the "access point sends a zeroed-or-omitted placeholder MIC" finding from [`mac-application-frames.md` § MAC header](mac-application-frames.md#mac-header) — that finding is about frames the *access point transmits*; this is a *device → access point* frame, which carries its own real security fields, passed through by the coprocessor to the host. This spec doesn't decode SecurityNumber's own semantics further.
- `85` — Application header byte 0: `ResponseRequested`=1, `StayAwake`=0, `FrameType`=5 (Status). The access point now owes this device an [Answer frame](glossary.md#answer-frame) (see Step 6).
- `02` — Application sequence number.
- `04 80 ...` — the Status frame's own payload: flags byte `0x04` (only `Booted` set), RSSI byte `0x80` (the device's own "no reading available" sentinel, not a real measurement — see [`mac-application-frames.md` § Status frames](mac-application-frames.md#status-frames) for why there are two unrelated RSSI values in this protocol), followed by opaque status data entries this spec doesn't decode further.

## Step 6 — Access point answers

*Verified on the wire — captured from the same real trace as Step 5,
immediately following it.*

Because Step 5's frame had `ResponseRequested` set, the access point
replies with an Answer frame, sent as an ordinary serial **Send protocol
frame** command (see [`serial-protocol.md` § HmIP command set](serial-protocol.md#hmip-command-set)):

```
00  10 00 8e  b1235a  2cf049  02 02  00
```

- `00` — burst mode: single transmission, no repeat strategy (see [`serial-protocol.md`](serial-protocol.md#hmip-command-set)).
- `10 00 8e` — MAC header: same `SecurityControl`=ENABLED flag as Step 5's frame, but — matching the "access point omits the security fields entirely and lets the coprocessor insert the real MIC" finding in [`mac-application-frames.md` § MAC header](mac-application-frames.md#mac-header) — **no SecurityNumber/IntegrityCode bytes follow at all** here; the Application header comes immediately after the addresses. `PiggybackAck`=0 this time.
- `b1235a` — MAC source address: the access point.
- `2cf049` — MAC destination address: the same device from Step 5.
- `02` — Application header byte 0: `ResponseRequested`=0, `StayAwake`=0, `FrameType`=2 (Answer).
- `02` — Application sequence number: the same value (`2`) as the request it's acknowledging.
- `00` — payload: answer type `0` = ACK.

See [`mac-application-frames.md` § Answer frames](mac-application-frames.md#answer-frames)
for the general shape this instance follows.
From here on, this device is in ordinary, fully-included operation —
everything in
[`native-device-links.md`](native-device-links.md) and
[`configuration-frames.md`](configuration-frames.md) now applies to it.

## What this walkthrough deliberately doesn't include

A second, smaller worked example (e.g. one native-link creation) was
considered and dropped: no other frame family in this spec has a
concrete, real captured byte example to build a second walkthrough from
— only field layouts and, for links, the real channel numbers involved
(not a full real byte sequence). Rather than invent plausible-looking
bytes, this page stops at the scenario the spec's existing real capture
actually supports.
