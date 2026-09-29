# Glossary

Short definitions for acronyms and protocol-specific terms used
repeatedly across this specification. Not meant to be read start-to-end —
look up a term as you encounter it elsewhere. Each entry links to the
file/section with the full explanation; this page deliberately doesn't
duplicate that material.

Terminology here matches this spec's own (de-vendorized) naming — plain
English labels for internal mechanisms, not the vendor's internal source
identifiers — consistent with how every other document in this repo
already names things.

### AAD (Additional Authenticated Data)

Extra data folded into a CBC-MAC computation so it's covered by the
resulting MIC without itself being encrypted. Whether AAD is present
changes which [B0](#b0) flag-byte value is used. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data)
(`calculate_mic`'s `authentication_data` parameter).

### Access point (AP)

The host-side implementation that manages HmIP devices through a
[coprocessor module](#coprocessor-module) — accepts device inclusions,
sends and answers Application frames, and (for cloud-mode inclusion)
talks to the [cloud key server](security-and-inclusion.md#cloud-key-exchange).
Used throughout this spec, most centrally in
[`security-and-inclusion.md`](security-and-inclusion.md).

### Answer frame

Application FrameType 2 — the acknowledgement a device or access point
sends back for a frame that had `ResponseRequested` set. See
[`mac-application-frames.md`](mac-application-frames.md#answer-frames).

### B0

The flag byte that forms the first block fed into the CBC-MAC
computation; its value depends on the MIC length (4 or 8 bytes) and
whether [AAD](#aad-additional-authenticated-data) is present. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data)
(`b0_flag_byte`).

### Burst mode

The 1-byte retransmission-strategy prefix on the serial "Send protocol
frame" command, chosen to match the target device's own declared
[listener mode](#listener-mode). Sending the wrong burst mode to a
cyclic/burst-listening device can silently fail to deliver the frame.
See [`serial-protocol.md`](serial-protocol.md#hmip-command-set).

### CBC-MAC

The MAC-computation half of this protocol's [CCM*](#ccm)-style
construction: message blocks (plus optional [AAD](#aad-additional-authenticated-data),
padded to 16 bytes) are chained through AES-128 starting from the
[B0](#b0) block; the raw output is then masked with an [S0](#s0)
keystream block to produce the final MIC. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

### CCM\*

The generic-composition, counter-mode-plus-CBC-MAC AEAD construction
this protocol's security layer is built from (standard AES-128 as the
only primitive). See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data)
("Underlying cryptographic primitives").

### ContentType

The MAC header's 4-bit field (byte 0, low nibble) selecting what kind of
payload follows — Application, Network Management, Route Management,
etc. See
[`mac-application-frames.md`](mac-application-frames.md#mac-header).

### Coprocessor module

The UART-attached radio hardware (e.g. RPI-RF-MOD / HM-MOD-RPI-PCB) that
performs the actual 868 MHz RF transmission and encryption; a host never
sees raw RF bytes or real key material for ordinary traffic, only
already-framed MAC frames. See [`README.md`](../README.md#architecture-at-a-glance)
and [`serial-protocol.md`](serial-protocol.md).

### Event listener

A [listener mode](#listener-mode) in which a device's receiver is only
active for a short window immediately after *it* transmits something —
typified by battery-powered window/door sensors. Any frame destined for
such a device can only be delivered during one of these windows. See
[`native-device-links.md`](native-device-links.md#sleeping-devices--the-real-constraint).

### FrameType

The Application header's 6-bit field identifying the frame's kind
(Configuration, Answer, Status, Direct Execution Command, etc.). See
[`mac-application-frames.md`](mac-application-frames.md#frame-types).

### Inclusion Accept frame

The access point → device frame that completes a device's inclusion,
carrying the joining device's new address and encrypted network key. See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-accept-frame-access-point--device).

### Inclusion Request frame

The device → access point frame a joining device re-sends periodically
(roughly every 10 seconds) until accepted, carrying its
[SGTIN](#sgtin), device-type identity, and the joining device's own key
material. See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-request-frame-device--access-point).

### Keystream

The CTR-mode-style pseudorandom block stream (`AES_encrypt(key, 0x01 ||
nonce || block_index)`), XORed with data to encrypt or decrypt it
symmetrically, and also used (at block index 0) to produce the
[S0](#s0) masking block for CBC-MAC output. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

### Link partner

The address (and channel) of the device on the other end of a native
device-to-device link. See
[`native-device-links.md`](native-device-links.md); for a device's own
configuration rather than a genuine link, this field instead carries the
["self" sentinel](#self-sentinel).

### Listener mode

A value in a device's own Inclusion Request frame declaring how/when its
receiver is reachable (e.g. single- or triple-burst listening, cyclic
variants of either, or [event listener](#event-listener)) — determines
both the correct [burst mode](#burst-mode) to transmit to it and, for
link/configuration writes, whether `StayAwake` must be set. Real observed
values, with device examples: `serial-protocol.md`'s listener-mode
table. See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-request-frame-device--access-point)
and [`serial-protocol.md`](serial-protocol.md#hmip-command-set).

### MAC frame

The actual HmIP protocol data unit — a MAC header, optionally followed
by an Application header and payload — carried as the data of a serial
"Send protocol frame" / "Received event" command. See
[`mac-application-frames.md`](mac-application-frames.md#mac-header).

### MIC (Message Integrity Code)

The output of the [CBC-MAC](#cbc-mac) computation (4 or 8 bytes),
authenticating a frame or an inclusion-handshake message. At the MAC
header level, a real access point relies on the coprocessor module
itself to compute and insert the real per-frame MIC rather than
computing it host-side. See
[`mac-application-frames.md`](mac-application-frames.md#mac-header)
and
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

### Network key

The per-device symmetric key established during inclusion (delivered to
the device encrypted, inside the Inclusion Accept frame) and used
thereafter for that device's ordinary secured traffic. See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-accept-frame-access-point--device).

### Nonce

A 13-byte value that, together with the shared key, parameterizes both
the [keystream](#keystream) and [CBC-MAC](#cbc-mac) computations for a
given exchange. For the network-key exchange step specifically, it's
derived from 9 bytes of the device's [SGTIN](#sgtin) plus a 4-byte
one-time value. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

### One-time key (OTK)

A random contribution each side generates fresh for a given inclusion
attempt, exchanged as part of the Inclusion Request / Inclusion Accept
frames. See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-request-frame-device--access-point).

### ResponseRequested

The Application header's 1-bit field indicating the sender expects an
[Answer frame](#answer-frame) acknowledgement. See
[`mac-application-frames.md`](mac-application-frames.md#application-header-contenttype--application).

### RSSI

Two independent, easily-confused values exist: the Status frame's own
RSSI byte (as measured by the *reporting device itself*, with `0x80` as
a "no reading" sentinel, not a real measurement), and the coprocessor's
own per-packet measurement of a received frame's signal strength,
delivered out-of-band as the leading byte of a serial "Received event"
command — the latter is what a real access point's UI typically shows as
"device RSSI". See
[`mac-application-frames.md`](mac-application-frames.md#status-frames)
and [`serial-protocol.md`](serial-protocol.md#hmip-command-set).

### S0

A keystream block computed at block index 0, XORed with the raw
CBC-MAC output to produce the final masked MIC. See
[`security-and-inclusion.md`](security-and-inclusion.md#access-point-side-key-material-get-inclusion-data).

### "self" sentinel

The reserved link-partner-address value (address type "plain device
address", all-zero 3-byte address, link-partner channel literal `0`)
used when a Configuration-family request targets a device's *own*
settings rather than a genuine link partner's. Using the target
channel number instead of literal `0` here is a real, easily-made
mistake. See
[`configuration-frames.md`](configuration-frames.md#the-self-link-partner-address).

### SecurityControl

The MAC header's 2-bit field (byte 0, bits 4–5) selecting whether, and
how, security fields (SecurityNumber, IntegrityCode) are present. See
[`mac-application-frames.md`](mac-application-frames.md#mac-header).

### SGTIN

The 12-byte device identifier carried in an Inclusion Request frame,
used (among other things) to derive the nonce for that device's
key-exchange and to look up its model via
[`device-catalog.md`](device-catalog.md). See
[`security-and-inclusion.md`](security-and-inclusion.md#inclusion-request-frame-device--access-point).

### StayAwake

The Application header's 1-bit field hinting to a battery-powered,
cyclic-listening device that it should keep its receiver open a little
longer than usual because the sender has more work queued for it — set,
for example, on an Answer frame when a configuration write is pending.
See
[`mac-application-frames.md`](mac-application-frames.md#application-header-contenttype--application).

### Verknüpfung

The vendor's own term (native-device-links.md's naming) for a native,
device-to-device link — e.g. a window sensor linked directly to a
thermostat, with no host/server involvement at runtime once the link
exists. See [`native-device-links.md`](native-device-links.md).
