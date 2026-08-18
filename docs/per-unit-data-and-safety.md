# Per-unit data, backing it up, and what can be damaged

Read this before pointing any software at a scanner. These scanners are out
of support, replacing one is expensive and getting harder, and each one
carries data that exists nowhere else. This page says what that data is, where it lives, how to copy
it out safely, what can be damaged and how, and what is recoverable. It
documents facts and procedure; it does not ship a tool.

_From the OEM service manual, the OEM engine's own read routine, live reads
of a real F-135+ (serial 16402), and incidents recorded by the independent
client projects, August 2026. Sources are cited per item._

## What is per-unit, and where it lives

| Store | Where | What | Replaceable? |
|---|---|---|---|
| **Boot personality** | I2C EEPROM at 7-bit `0x51` on the motherboard, read by the FX2 bridge silicon at power-up | 9 bytes: `c0 05 0f 35 f2 07 aa 04 02` on an F-135/F-135+ (C0-format load: VID `0f05`, PID `f235`, revision `aa07`, config byte, one trailing byte). Selects the cold USB identity and so which firmware image the host loads. | **Yes.** The bytes are known per model and Kodak shipped them as files (`FirmwareLoader/Personalities/USB F135.bin`). |
| **Per-unit EEPROM** | I2C EEPROM at 7-bit `0x52` on the motherboard, read by the host through the bridge | Serial number; per-resolution `Offset` (the OEM's name; most likely the CCD pixel-window start for that base, see [calibration.md](calibration.md#the-per-unit-eeprom)) and motor speeds (normal and IR); motor-adjust words; the two 3x10 colour matrices. Stored twice with CRC-32. See [calibration.md](calibration.md#the-per-unit-eeprom). | **No.** Measured at the factory for this unit's optics and transport. |
| **Light calibration** | The host's registry (`HKLM\Software\Pakon\TLB\Scan\DpiBase<N>_35\<mode>`), written by the OEM engine | LED currents, duty cycles (open-gate and with-film), A/D gains and offsets, per resolution base and film mode. | **Yes.** Derived by running the OEM's Light Correction (or an equivalent measurement); LEDs age, so it is meant to be re-derived. |
| **PIC firmware** | Flash in the light and motor controllers (PICL/PICM, PICL+/PICM+) | Kodak's controller firmware, per hardware revision. | **Yes, with care.** Reflashable from the OEM images, but only the image matching the board's hardware revision (the OEM readme warns in capitals not to use `03`/`04`/`05` images on PCB 125039A). |
| FX2 firmware | RAM in the USB bridge, uploaded by the host at every power-up | `Pakon7.hex` and siblings | Not per-unit and not persistent; a power cycle discards it. |

[CONFIRMED] for the two EEPROMs by live reads on serial 16402 and by
[pakon-mac](https://github.com/gazzdingo/pakon-mac)'s I2C bus survey; the light-calibration location from the OEM
engine's registry writes and its own documentation. Note the service manual's
"the EEPROM is the source of truth and the registry a cache" is true only of
what the EEPROM holds; for light calibration the registry is the only copy.

## What can be damaged, and how

Everything below has happened to a real scanner or has been demonstrated
on one. None of it happens by reading; every item is a write, or a
malformed packet.

- **Erasing PIC firmware over the ordinary command channel.** The
  controllers' bootloaders answer on their own bus addresses (`0x22`/`0x26`
  on the F-135, `0x42`/`0x46` on the F-135+), on the same command channel
  as everything else. A type-4 packet to a bootloader address whose command
  byte has bits 3 and 2 set (`0x0C`–`0x0F`) is dispatched as a 64-byte flash
  row erase at the latched address. A real unit lost a row of its motor
  controller's firmware this way. [CONFIRMED] by that incident (pakon-mac,
  [`tools/probe_picm_alive.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/probe_picm_alive.py#L20-L21))
  and by the bootloader protocol as read from the OEM loader (pakon-tlx-macos,
  [`README.md`](https://github.com/pablonavarrob/pakon-tlx-macos/blob/3e3fa0fa830e/README.md#L284-L289) and
  [`server/pakonusb.py`](https://github.com/pablonavarrob/pakon-tlx-macos/blob/3e3fa0fa830e/server/pakonusb.py#L120)).
  Links are pinned to the commits reviewed.
  Recovery required reflashing the controller from the OEM image.
- **Erasing the boot personality by writing to unknown bus addresses.** The
  PPB bus addresses are I2C addresses shifted left one bit; addresses that
  are not a known controller may be I2C devices, including the two EEPROMs
  (`0xA2` = boot EEPROM at `0x51`, `0xA4` = per-unit EEPROM at `0x52`). A
  sweep of writes across assumed board addresses erased a real unit's boot
  personality; it then enumerated as a bare FX2 (`04b4:8613`) with a red
  status LED and needed an explicit firmware image at every power-up until
  the 9 bytes were rewritten. [CONFIRMED] by that incident (pakon-mac,
  [`tools/eeprom_dump.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/eeprom_dump.py#L4-L6) and
  [`backups/eeprom-i2c/README.md`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/backups/eeprom-i2c/README.md)).
- **Writing the per-unit EEPROM.** Vendor control request `0xA2` (the write
  counterpart of the `0xA9` read) writes the `0x52` chip. The OEM's scanning
  path never issues it; only its Calibration Wizard does. A wrong write here
  destroys data that cannot be regenerated. [DOCUMENTED] from the OEM
  engine; no incident on record, and the point of this page is that there
  should not be one.
- **Wedging the FX2 bridge with a malformed packet.** A packet whose type
  byte is `0` is not accepted and leaves the firmware unable to drain its
  command endpoint; the state survives a USB reset because the firmware is
  in RAM and a reset does not restart it. Only a power cycle clears it. An
  invalid *payload* is harmless (status 2); an invalid *type* is not.
  [CONFIRMED] (pakon-mac, [`docs/03-protocol.md`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/docs/03-protocol.md#L91-L96)).
  Recoverable, but a session-ending surprise.
- **Loading the wrong FX2 firmware image.** The personality byte read
  through the stage-1 loader can be unstable immediately after that loader
  starts; selecting the image from it alone once loaded F-235 firmware onto
  an F-135, which lit a fault LED. Recoverable with a power cycle (firmware
  is RAM-resident), but avoid it: cross-check against the cold USB identity.
  [CONFIRMED] by that incident (pakon-mac,
  [`tools/pakon_load.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/pakon_load.py#L236-L241)).
- **Driving LEDs above the board's ceiling.** The controllers enforce
  per-channel LED-current maxima that differ by board (`0x24`: R 6, G 8, B 8;
  `0x44`: R 4, G 20, B 20 with IR off, higher with IR on). Applying the
  `0x44` table to a `0x24` board permits about 2.5x the vendor's own limit
  on two channels. Whether that damages the LEDs is not established; the
  vendor's ceilings exist for a reason. [DOCUMENTED] from the OEM engine
  (pakon-tlx-macos, [`docs/PROTOCOL.md`](https://github.com/pablonavarrob/pakon-tlx-macos/blob/3e3fa0fa830e/docs/PROTOCOL.md#L200-L206)).

What is **not** at risk: the FX2 firmware (RAM; a power cycle restores the
cold state), and anything on the host. Reading, polling and PPB commands to
the known controller addresses have no recorded incident.

## Backing up the per-unit EEPROM

The safe way is to issue exactly the transfers the OEM engine itself
issues when it reads this chip at every launch, and nothing else. Those
transfers are known to be serviced by the vendor firmware, they are reads,
and they put no code on the scanner. Any tool that does this is replaying
the OEM's own sequence; the steps below spell it out so a tool can be
checked against it:

1. Get the scanner operational (firmware loaded, `0f05:f135`), and make sure
   nothing else holds the USB device.
2. Issue only the two read requests, per 32-byte chunk: vendor OUT `0xA4`
   with `wValue 0x00A5`, `wIndex 0x1234`, no data (select the chip for
   reading); then vendor IN `0xA9` with `wValue` = byte offset,
   `wIndex 0x1234`, `wLength` ≤ 32. Never issue `0xA2`.
3. Read **both copies of both sections**: A at `0x000` and `0x400` (398
   bytes each), B at `0x800` and `0xA00` (36 bytes each), each by its own
   `{u32 length; u32 crc32}` header.
4. Check all four CRCs (CRC-32, zlib, over the payload) and compare each
   primary with its backup byte for byte. A dump of the primaries alone
   cannot show whether it is good: the reference unit carries one corrupted
   byte in its primary section A that only the CRC and the backup reveal.
5. Read once per power cycle, then confirm by comparing dumps across power
   cycles. The [pakon-mac](https://github.com/gazzdingo/pakon-mac) project
   reports that on its unit these EEPROMs returned good data only on the
   first transaction after power-up and degraded on later reads while still
   reporting success
   ([`backups/eeprom-i2c/README.md`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/backups/eeprom-i2c/README.md),
   [`tools/eeprom_oneshot.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/eeprom_oneshot.py#L5-L18)).
   That is not how these parts normally behave and may be specific to that
   unit or its tooling; the reference unit shows no such behaviour over four
   reads. Until it is understood, reading once per power cycle costs
   nothing and avoids the question.
6. Sanity-check the decode: the serial number at `0x010` should be your
   unit's; section A's length field is 398, section B's is 36; PosMatrix
   should be a plain 0.25 diagonal.

Keep the raw bytes and their hashes somewhere that will outlive the
machine you read them on. An existing implementation of exactly this
sequence, read-only, is [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos)'s
`tools/eedump.py` (with the backup and CRC checks proposed in its
[PR 4](https://github.com/pablonavarrob/pakon-tlx-macos/pull/4));
[pakon-mac](https://github.com/gazzdingo/pakon-mac)'s `eeprom_oneshot.py`
reads the same chips a different way: it loads its own small FX2 firmware
into the bridge's RAM, which drives the I2C bus directly. That works and
produced their verified backups, but it is the riskier route: it puts
third-party code on the scanner (the FX2 is unbrickable, RAM only, but that
code is bit-banging the very bus the two EEPROMs sit on), and its safety
rests on reading that firmware's source rather than on replaying transfers
the OEM already makes at every launch. Prefer the OEM's own read; use the
custom-firmware route only if the OEM path is unavailable, and read its
source first. Do not use tools
that guess at the request parameters: a read with the wrong `wValue`/
`wIndex` addresses no EEPROM and returns bus-idle `0xFF` while looking
like data.

## Backing up the boot personality and the light calibration

- **Boot personality (`0x51`):** the same two requests read it (select
  `wValue 0x00A3` = `(0x51 << 1) | 1`, then `0xA9` at offsets 0–8), or
  simply record that your unit's cold identity is `0f05:f235` revision
  `aa07` and the bytes are the known F-135 personality. If it is ever
  erased, those 9 bytes are what to write back.
- **Light calibration:** export the registry tree `HKLM\Software\Pakon\TLB`
  (or the `Wow6432Node` twin) after the OEM has run Light Correction. It is
  re-derivable, but keeping it saves a calibration run and records the
  values your unit settled on. The OEM also mirrors the per-unit EEPROM
  into the same tree, which is useful as the OEM's own decode to check yours
  against, but it is not a substitute for reading the chip: it holds only
  the copy the OEM ended up using, already resolved (a unit with a corrupt
  primary looks perfectly healthy in the registry), as decimal strings that
  lose the last bits of the floats, without the headers, CRCs, or unused
  bytes; and it exists only where the OEM software has run against the
  scanner. Back up the raw chip; keep the registry export for the light
  values and as a cross-check.

## Recovery

| Lost | Recover by |
|---|---|
| Boot personality | Write the 9 known bytes back to the `0x51` EEPROM (an FX2 write, not the `0x52` chip). Until then the unit enumerates as `04b4:8613` and needs an explicit image; the scanner otherwise works (pakon-mac's unit runs this way: [`tools/eeprom_repair.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/eeprom_repair.py#L11-L23), [`docs/01-usb-layer.md`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/docs/01-usb-layer.md#L33-L48)). |
| Per-unit EEPROM, one copy | Nothing to do: the OEM reads the backup when the primary's length or CRC is bad, first good copy wins. Repairing the primary needs an `0xA2` write; weigh it against the risk. |
| Per-unit EEPROM, both copies | Only from a backup you made. There is no other source. |
| Light calibration | Run Light Correction (or an equivalent measurement) again. |
| PIC firmware row | Reflash the controller from the OEM image for that board's hardware revision, over the bootloader protocol. This has been done once on a real unit, by pakon-mac, whose [`tools/flash_picm.py`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/tools/flash_picm.py) is the implementation (its detailed recovery notes are on that project's private remote, per its [`docs/68-handover.md`](https://github.com/gazzdingo/pakon-mac/blob/c0be5853c292/docs/68-handover.md#L36-L44)). The bootloader protocol is not documented in this reference yet; it is delicate, and the wrong revision damages the board. |
| Wedged FX2 | Power cycle. |

## Open questions

- Whether sustained LED current above the board's ceiling causes lasting
  damage, and whether that differs between the F-135 and the F-135+: the
  Plus has a thermoelectric cooler and temperature sensing on the light board
  (the `0xD0`/`0xD1` init and the `0x84`/`0x88` temperature reads), the base
  F-135 does not, so the same over-current may be tolerated on one and not
  the other.
- Whether the EEPROM contents can really become unstable when read more
  than once within a single power cycle, as one project reports of its unit
  (later reads differing from the first while still reporting success), and
  if so why; the reference unit reads consistently. Until it is understood,
  reading once per power cycle sidesteps it.
- The boot EEPROM's contents beyond the 9-byte personality.
