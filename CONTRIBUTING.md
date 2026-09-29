# Contributing

This specification improves by comparing it against real traffic from real
hardware. This document describes a proven, low-overhead technique for
independently capturing that traffic, verifying (or correcting, or
extending) any part of this spec against it, and contributing the result
back.

## Purpose

If you have your own radio coprocessor module, you can independently
confirm — or refute — any claim in this spec, resolve items on
[`docs/known-gaps.md`](docs/known-gaps.md), and capture scenarios nobody
has recorded yet. The method below is not hypothetical: it is how several
of the "verified on the wire" findings elsewhere in this spec were
actually established.

## Prerequisites

- A compatible radio coprocessor module, attached to a host via UART.
- At least one real HmIP end device to pair and/or interact with — a
  trace with no device activity in it confirms very little.

## Transparent serial relay + capture

The core technique is a passive, byte-exact relay inserted between the
real serial device and whatever software you want to observe, with every
byte logged in both directions. Below is a complete, minimal, working
example (Python 3, standard library plus [`pyserial`](https://pyserial.readthedocs.io/)
for the real-device side) — adapt paths/baud rate to your own setup.

### 1. Get exclusive access to the serial device

Stop any process already holding it open — the relay needs the port to
itself. Find out what's using it, then stop that specifically (prefer
this over anything more forceful):

```sh
# find what's holding the port open
lsof /dev/ttyUSB0            # or: fuser -v /dev/ttyUSB0

# if it's a systemd-managed daemon, stop it cleanly
sudo systemctl stop <the-service-name>

# otherwise, stop the specific process lsof/fuser reported
sudo kill <pid>
```

### 2. Create a PTY pair and put the slave into raw mode

**Easy-to-miss gotcha:** a freshly created PTY defaults to
canonical/line-buffered terminal mode with echo enabled — meant for a
human typing at a terminal, not transparent binary passthrough. Left in
that default mode, it will silently corrupt binary protocol traffic. The
concrete symptom is bytes that look like garbled, duplicated, or
interleaved fragments of what should be clean protocol frames —
confusing to debug if you don't already know to look for it. Fix: put
the PTY slave into raw mode immediately after creating it, before
anything opens it — the standard library's `tty.setraw()` does exactly
this (clears the canonical/echo/signal-processing terminal flags in one
call).

```python
import os
import pty
import tty

master_fd, slave_fd = pty.openpty()
tty.setraw(slave_fd)                    # <- the gotcha fix; do this first
slave_path = os.ttyname(slave_fd)

# PTY slave paths (/dev/pts/N) are not stable across runs - symlink to a
# fixed path so whatever you point at it doesn't need updating every time
symlink_path = "/tmp/hmip-relay-serial"
if os.path.lexists(symlink_path):
    os.remove(symlink_path)
os.symlink(slave_path, symlink_path)
print(f"relay ready at {symlink_path} -> {slave_path}")
```

### 3. Point the software you want to observe at the symlinked path

Your own implementation under test, or optionally a complete reference
implementation of the host-side software if you have access to one —
configure it to use `/tmp/hmip-relay-serial` (from step 2) *instead of*
the real device path.

### 4. Forward bytes bidirectionally and log every chunk

Continuing the same script — open the real serial device, then relay in
both directions, logging each chunk with a timestamp, a direction
marker, and the raw bytes as hex:

```python
import datetime
import queue
import threading

import serial  # pip install pyserial

LOG_PATH = "hmip-capture.log"

# Logging happens on a dedicated background thread, fed by a plain
# thread-safe queue - relaying always happens first, logging never blocks
# it. An earlier version of this relay opened/wrote/closed the log file
# synchronously inline in each pump function; under a fast back-to-back
# exchange (e.g. a device inclusion handshake), that real disk I/O was
# enough added latency, on the hot path, to matter. It turned out not to
# be the actual cause of a real timeout bug chased for hours one night
# (see git history/known-gaps.md for that story) - but it's still a real
# latency source worth avoiding on principle, and costs nothing to fix.
_log_queue: "queue.SimpleQueue[tuple[str, str, bytes] | None]" = queue.SimpleQueue()


def _log_writer() -> None:
    with open(LOG_PATH, "a", buffering=1) as f:
        while True:
            item = _log_queue.get()
            if item is None:
                return
            ts, direction, data = item
            f.write(f"{ts} {direction:16s} {data.hex()}\n")


def log(direction: str, data: bytes) -> None:
    ts = datetime.datetime.now(datetime.timezone.utc).isoformat()
    _log_queue.put_nowait((ts, direction, data))


def pump_device_to_pty(device: serial.Serial) -> None:
    while True:
        data = device.read(1)                      # blocks until >=1 byte
        data += device.read(device.in_waiting)      # drain whatever else has arrived
        if data:
            os.write(master_fd, data)               # relay first
            log("module-to-host", data)


def pump_pty_to_device(device: serial.Serial) -> None:
    while True:
        data = os.read(master_fd, 4096)             # blocks until >=1 byte
        if data:
            device.write(data)                      # relay first
            log("host-to-module", data)


threading.Thread(target=_log_writer, daemon=True).start()
device = serial.Serial("/dev/ttyUSB0", baudrate=115200, timeout=None)
threading.Thread(target=pump_device_to_pty, args=(device,), daemon=True).start()
threading.Thread(target=pump_pty_to_device, args=(device,), daemon=True).start()
print(f"relaying {device.port} <-> {symlink_path}, logging to {LOG_PATH} - Ctrl+C to stop")
threading.Event().wait()   # block forever
```

Run steps 2–4 as one script (they share `master_fd`/`slave_fd`/`symlink_path`),
start the software from step 3 pointed at the printed symlink path, and
let it run for whatever scenario you're capturing. The result is a
byte-exact, timestamped trace of everything crossing the wire in both
directions — for example:

```
2026-09-25T20:22:45.080635+00:00 host-to-module   fd000a02a4045f1540040000004f9d
2026-09-25T20:22:45.090558+00:00 module-to-host    fd000402a4060135ee
```

(A real correlated pair, not a fabricated illustration — a real "Add link
partner" call and its Response, both fully self-consistent with their
own declared `Length` field: see `docs/security-and-inclusion.md`'s
Inclusion Accept worked example for the decoded fields.)

This is non-destructive: the relay is a passive pass-through that never
alters the traffic, only observes and forwards it.

## Ground-truth capture via a reference implementation (optional but powerful)

If you have access to a complete, independently-known-working reference
implementation of the host-side software — for example, runnable in a
container against the same physical module — point it at the PTY instead
of your own implementation under test for a given scenario (pairing, a
specific command, a specific frame type). This gives you verified,
correct reference traffic to diff your own implementation's behavior
against byte-for-byte. This is a materially stronger verification method
than reasoning from decompiled logic alone, and is how several of the
"verified on the wire" claims elsewhere in this spec were actually
established.

**Docker gotcha #1 if the reference implementation runs in a container:**
don't pass the PTY slave in with `--device host-path:container-path`.
Docker gives every container its own private devpts mount namespace, so
a `--device` char-device passthrough mknod's a node with the right
major:minor number, but the container's kernel-side pty lookup resolves
opens through its *own* devpts instance rather than the host's — the
node looks identical (same permissions, same major:minor) but `open()`
on it fails with `EIO` from inside the container, while the exact same
open() succeeds fine from the host. The fix that actually works: bind-mount
the host's whole `/dev/pts` into the container (`-v /dev/pts:/dev/pts`) so
it shares the host's devpts instance, **and** bind-mount the specific PTY
slave path directly at the expected device path (`-v
/dev/pts/N:/dev/ttyUSB0`, or whatever path the reference software expects
— plain `-v`, not `--device`). Don't rely on a runtime-created symlink
*inside* the container for this (see gotcha #2 below for why) — pass the
device path as a real bind mount at container-creation time.

Note that a real physical device (not a PTY) doesn't have this problem at
all — a plain `-v host-device:container-path` bind mount is enough by
itself for those, since there's no devpts-instance concept involved. This
gotcha and its fix are specifically about PTYs.

Also note: a real physical device bind-mounted this way still needs
`--device` (not just `-v`) if you want the container's device-cgroup
policy to actually grant access to it — `-v` alone makes the file
*visible* inside the container, but a plain bind mount doesn't add the
cgroup allow-rule a non-standard character device needs; without it,
`open()` fails with `EPERM` even as root inside the container. (A PTY's
own major:minor is already covered by the `/dev/pts` bind mount from
gotcha #1, so this doesn't apply there — it's specifically for the real,
physical serial device if you go the gotcha-#1-doesn't-apply route.)

**Docker gotcha #2, specific to a JVM-based reference implementation
(observed with a real vendor's Java serial library, `gnu.io`/RXTX-family):**
some serial libraries snapshot their own internal list of "known" ports
*once*, early in JVM startup — via a static initializer that scans `/dev`
or consults a configured device-list property at that exact moment. If
the expected device path doesn't exist yet right then (a real risk on a
slow boot, e.g. right after a from-scratch container start doing a lot of
other first-time initialization), the library permanently excludes it for
that JVM's entire lifetime — every later attempt to open it fails with
some form of "no such port" exception, even though the path demonstrably
exists and is openable via every other means at that later point. This
showed up as a **genuinely intermittent** failure: the identical setup
(device present via a bind mount from gotcha #1, before the JVM ever
starts) succeeded on most attempts and failed on a few, seemingly at
random. The one fix that reliably recovered it: a **full, clean container
restart** (`docker restart`, exercising the whole boot sequence fresh, not
just restarting the reference software's own process/service in
isolation) — restarting only the affected process while everything else
around it stays up did *not* reliably clear it, but a full restart did,
every time it was tried, sometimes needing one retry.

## Bracketing individual actions in a long capture session

If you're capturing several distinct, deliberate actions in one long
session (several link creations, a config write, a few different `setValue`
calls) rather than one continuous scenario, insert a plain marker line
into the log immediately before and after each action — something that
can't be mistaken for a real captured frame (all of this spec's own real
captures used `===== BEGIN: <what you're about to do> =====` /
`===== END: <outcome> =====`, but the exact format doesn't matter, only
that it's unambiguous). This costs nothing and pays off immediately once
you're decoding a session with more than two or three actions in it —
without it, correlating "which bytes belong to which RPC call" after the
fact from timestamps alone gets error-prone fast, especially once you
have several actions with real acks and device replies interleaved
within seconds of each other. Note the *outcome* in the END marker too
(succeeded / faulted / inconclusive, and how you confirmed it — e.g. "via
a follow-up read-back call") — that context is easy to lose by the time
you sit down to decode the trace, and is exactly what tells you which
sections are worth spending decoding effort on.

## Sleeping devices need a wake trigger for a pending command, not just for capture

[`docs/native-device-links.md`](docs/native-device-links.md)'s "Sleeping
devices" section already covers this for *ordinary application traffic*,
but it's easy to forget it also applies when you're the one issuing a
command via RPC during a capture session: an event-listener-class device
(no periodic listening at all — typically a battery-powered sensor) won't
receive *any* command queued for it — including something you triggered
yourself, like a device-removal/unpair call — until the device itself
transmits something and briefly opens a receive window. A command queued
for a sleeping device will sit reporting some form of "pending" or
"transmission in progress" indefinitely; physically triggering the
device (toggle a contact sensor, wake a button) is what actually delivers
it. This isn't a bug or a stuck state to debug around — it's the real
protocol behavior, and it directly shapes how you capture the byte-level
frames for pending-command delivery: expect a wait, not an immediate
result, for anything targeting this class of device.

## What's most valuable to capture

Use [`docs/known-gaps.md`](docs/known-gaps.md) as the priority list — an
item there that your capture settles is the most valuable kind of
contribution. Beyond that:

- A device being reset and freshly paired — the whole inclusion handshake
  in one continuous trace.
- Routine status traffic over a few minutes of otherwise-idle operation.
- Most valuable for correlating specific bytes to specific real-world
  effects: a trace spanning a deliberate state change you trigger
  yourself (toggling a sensor, changing a setpoint), noted with exactly
  when and what you did, so the resulting frames can be matched to the
  action.

## Contributing a capture back

Open a pull request with the raw hex trace plus your own annotated
interpretation of it, stating clearly which existing doc/section it
confirms, corrects, or extends — or, if it doesn't match anything already
written, that it establishes something new. Per-device key material in a
trace is specific to your own device and isn't exploitable against
anyone else's, so there's no need to redact it for that reason.

**Do redact your device SGTINs before posting a capture, though.** A
joining device's Inclusion Request frame (and anything derived from it,
like a Get-inclusion-data request) carries its full SGTIN — a real,
globally unique identifier printed on the device and its retail
packaging, in the same spirit as a serial number or a MAC address. A raw
capture with an unredacted SGTIN is, in principle, an identifiable
component tied to a specific physical unit you own. Before sharing:
obscure at least the last few hex digits (this spec's own examples
redact the SGTIN's last 4 hex digits, e.g. `...898cxxxx`) — enough that
the value stops being a stable, traceable identifier, while every other
byte in the capture stays exactly as observed. This doesn't apply to
device *addresses* (the 3-byte MAC-layer addresses assigned during
inclusion) — those are short-lived, network-local values, not tied back
to a specific physical unit the way an SGTIN is.

## Safety and etiquette

The relay itself is passive and read-only, and safe to leave running.
Pairing or resetting a physical device, on the other hand, is an action
on your own hardware with real effects — it can un-pair a device from
whatever system currently controls it. Do this deliberately, not
accidentally, especially if the device is in active use.
