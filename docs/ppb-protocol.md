# The PPB command protocol

The F-X35 scanners are driven over USB 2.0 by a command protocol the OEM
software calls **PPB** (a parallel-bus bridge model). A bridge chip on the
scanner presents itself to the host and relays framed commands to the internal
microcontrollers; despite the USB transport, the protocol is modelled as a
host addressing devices on a bus.

_Established from analysis of the OEM Windows driver (TLA.dll) and USB captures
of real hardware, March 2026._

## Bus participants

The host addresses several devices by a one-byte bus address:

| Address | Device | Present on |
|---|---|---|
| `0x10` | HOST: the bridge itself | all models |
| `0x20` / `0x22` | PICL / PICL bootloader: light + CCD controller | F-135 |
| `0x24` / `0x26` | PICM / PICM bootloader: motor controller | F-135 |
| `0x40` / `0x42` | PICL+ / bootloader: light + CCD (Plus) | F-135+ |
| `0x44` / `0x46` | PICM+ / bootloader: motor (Plus) | F-135+ |

[DOCUMENTED] from the OEM driver's address constants and observed in captures.
The F-235/F-335 use a different multi-board architecture; their bus addressing
is not documented here.

The `0x20`/`0x24` vs `0x40`/`0x44` split is the central difference between the
F-135 and F-135+ command streams: the same commands, sent to different
addresses. [INFERRED] from the paired address constants; the two controllers
are otherwise driven identically in captures.

## Frame format

A command frame is:

```
[type] [count] [data ...]
```

- `type`: one byte selecting the operation (below).
- `count`: the number of data bytes that follow.
- `data`: `count` bytes; `data[0]` is the destination bus address, and for
  command/write frames `data[2]` is the command byte.

The on-wire length is `2 + count`. There is **no checksum** and no padding to a
fixed size; the protocol relies on USB's own integrity. [CONFIRMED] from
captures: the frame length always equals `2 + count`, and the byte that might
look like a checksum varies independently of the payload (it is a command
parameter).

### Type byte values

| Value | Name | Meaning |
|---|---|---|
| `0x01` | READ | host reads N bytes back from a device |
| `0x02` | WRITE | host writes N bytes to a device register/buffer |
| `0x03` | READ_STATUS | one-byte status poll (no command byte) |
| `0x04` | CMD | a command with no data payload |
| `0x07` | ACK | a device→host acknowledgement reply (to CMD and WRITE) |

[DOCUMENTED] `0x04` on host commands and `0x07` on their acknowledgements are
observed directly in the open-handshake captures; the WRITE/READ/READ_STATUS
values are read from the OEM driver's IOCTL encoding. Note that `0x07` is not
a universal reply type: replies to READ and READ_STATUS echo their own request
type (see below).

### Reply frames and the status byte

Replies take three forms, keyed by the request type:

| Request | Reply form |
|---|---|
| CMD (`0x04`) or WRITE (`0x02`) | ACK: `07 02 <addr> <status>` |
| READ_STATUS (`0x03`) | `03 02 <addr> <status>` |
| READ (`0x01`) | `01 <count> <addr> <status/flags> <data…>` |

In each, `data[0]` echoes the address and the following byte carries a status:

| Status | Meaning |
|---|---|
| `0x00` | success / acknowledged |
| `0x01` | not acknowledged (device absent or command rejected) |
| `0x02` | invalid packet |
| `0x03` | bad checksum |
| `0x04`–`0x06` | USB-layer errors |
| `0x07` | host-algorithm error |
| `0x08` | also reported as success in some sequences |
| `0x09` | bus error |

[DOCUMENTED] from the OEM driver's status enumeration. A reply too short to
contain a status byte carries no status; it must not be read as success.

### Reply structure, recovered by driving the scanner

The reverse-engineering captures were **transmit-only** (the OEM driver logs
recorded host→device frames but not the device's replies), so the exact bytes
of the reply direction were unknown until open-source code drove a real scanner
and read them back. [CONFIRMED on hardware, August 2026, in the pakon-macos
project by Jorge Rangel: https://github.com/jorshhh/pakon-macos]

- Reply types mirror the request: READ replies come back with type `0x01`
  (`01 <count> <addr> <status/flags> <data…>`), status polls with `0x03`, and
  CMD/WRITE acknowledgements with `0x07`.
- The byte after the address in a READ reply is a **status/flags byte**, not a
  constant: `0x08` was observed as the ordinary value and `0x88` when an event
  was pending, consistent with `0x80` being an event flag OR-ed onto `0x08`.
  Its complete bit semantics are not yet established.
- A `03 01 <addr>` status poll returns `03 02 <addr> <status>`; the value
  `0x80` was observed meaning "host event pending".
- The bridge replies to the HostReset/HostSetMode frames of the open handshake
  **only on the first open after power-on or firmware load**; on later opens
  those replies time out while the bridge keeps working normally. A driver
  should treat those two replies as best-effort, not as failures.

## The open handshake

A session begins by resetting the bridge and probing which controllers are
present:

```
host → 04 03 10 00 85        (HostReset to 0x10)      → 07 02 10 00
host → 02 04 10 01 8f 00     (HostSetMode to 0x10)    → 07 02 10 00
host → 04 03 <picm> 00 00    (probe a motor PIC)      → 07 02 <picm> <status>
...  (further presence probes)
```

[DOCUMENTED] verbatim from the open-handshake capture. The HostReset/HostSetMode
frames are the bridge open; PIC-directed traffic only flows after them.

### Presence probes are model detection

The probes that follow the bridge open are the OEM driver's **model
detection**: it asks whether a controller answers at each candidate address.
A reply status of `0x00` means that controller is present; `0x01` means
absent. Because the F-135 and F-135+ place their controllers at different
addresses, the two models answer these probes with opposite results. This is
how the OEM tells them apart. [CONFIRMED on hardware, August 2026] a real
F-135+ answers the Plus motor address (0x44) present and the F-135 address
(0x24) absent, exactly inverted from an F-135, verified by the pakon-macos
project by Jorge Rangel: https://github.com/jorshhh/pakon-macos, driving the
scanner. (The reply bytes and the open handshake's first-open-only reply timing
are detailed under "Reply structure" above.)

## Command channel

Commands ride a bulk endpoint pair, not USB control transfers: endpoint
`0x01` OUT carries the host→device frame and `0x81` IN carries the reply, as an
atomic write-then-read. A separate bulk IN endpoint carries the image stream
(see [image-stream.md](image-stream.md)). [DOCUMENTED] from the operational
device's endpoint descriptor and scan captures.

A structured parameter/calibration table is read separately over USB control
transfers (vendor requests `0xA4`/`0xA9`) in fixed-size chunks. [DOCUMENTED]
from captures; contents detailed in [calibration.md](calibration.md).

## Open questions

- Reply payloads for most commands are unobserved (the captures were
  transmit-only; only the exchanges exercised live are confirmed).
- The complete bit semantics of the READ reply status/flags byte.
- The `0xA4`/`0xA9` parameter table's field layout.
- Whether the F-235/F-335 use these bus addresses at all.
