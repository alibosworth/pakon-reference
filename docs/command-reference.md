# Command reference

The commands the host sends to each controller, and the sequences they compose
into. Command bytes are shared numbers interpreted per destination address: a
given byte means different things to the light controller and the motor
controller, disambiguated by the frame's address.

_Reconstructed from a large PPB debug trace of a real scanner and from the OEM
driver's API strings, March 2026._

A marker note that applies to every table below: the command **bytes, types,
payload sizes, and sequencing** are [DOCUMENTED] from the trace. The command
**names are largely [INFERRED]**: some match OEM API property strings, but
most were assigned from payload size, occurrence counts, and position in the
sequences, and the source analysis itself labels them inferred. Treat every
name as a working label, not an OEM identifier, unless noted otherwise.

Rows marked **(TLB)** are named from the OEM engine, `TLB.dll` 3.1.0.28
(sha256 5866ec56…), November 2026, via open-pakon
(https://github.com/jorshhh/open-pakon). Old labels are in the changelog.

See [ppb-protocol.md](ppb-protocol.md) for the frame format these ride in.

## Light / CCD controller (PICL, address 0x20 / 0x40)

| Cmd | Type | Payload | Name | Notes |
|---|---|---|---|---|
| `0x80` | WRITE | 1 | LedEnable (TLB) | bit 0 visible LEDs, bit 1 IR LED; `00` all off |
| `0x81` | WRITE | 5 | SetLedCurrents (TLB) | one byte each, order B, IR, R, 0, G; never above the board's ceiling |
| `0x82` | WRITE | 12 | SetLedDuties (TLB) | u16 LE: duty B, IR, R, 0, G, then the period; each duty ≤ period − 2 |
| `0x83` | READ | 1 | ReadLampFlags (TLB) | `0x02` lamp stable, `0x20` warning, `0x40` fault |
| `0x84` | READ | 2 | ReadLampSetpoint (TLB) | lamp temperature setpoint, 0.0625 °C per count |
| `0x87` | WRITE | 2 | (unknown) | `00 00` at init; not light power, which is `0x81`/`0x82` |
| `0x88` | READ | 4 | ReadTemperature | temperature sensors |
| `0x89` | WRITE | 1 | ResolutionFlag (TLB) | 1 at Base 8 (narrows the line), else 0 |
| `0x8A` | CMD | 0 | ResetFifos (TLB) | second half of a FIFO reset, after host `0x84` = `02`. Never during a live scan stream |
| `0x8B`/`0x8D` | WRITE | 4 | Board temperature bands (TLB) | warning / fault; two s16 LE words, low then high, 0.0625 °C |
| `0x8F`/`0x8C` | WRITE | 4 | Lamp temperature bands (TLB) | warning / fault, relative to the setpoint; same encoding |
| `0x90` | READ | 30 | ReadSensorData | DX/position entries; layout in [dx-barcode.md](dx-barcode.md#how-the-host-reads-it). Read only after the controller raises its service flag [CONFIRMED] |
| `0x91` | WRITE | 3 | SetScanLineParams | scan-line/resolution parameters; also resets the position counter reported by `0x90` [CONFIRMED] |
| `0x92` | CMD | 0 | EndAcquisition | end CCD acquisition; closes the DX decode window [CONFIRMED] |
| `0x93` | READ | 4 | ReadDxSensors | one byte per DX photodetector; polled at ~3 ms by the Film Track Test [CONFIRMED live on an F-135+] |
| `0xD0`/`0xD1` | WRITE | 1 | SetTEC 1/2 | thermoelectric cooler; F-135+ only |

`0xD0`/`0xD1` (TEC control) appear only on the F-135+, whose CCD is cooled.
[DOCUMENTED from the trace]

> **Caution: TEC control could potentially damage hardware.** The `0xD0`
> (setpoint) and `0xD1` (enable/mode) byte semantics are not understood, and it
> is unknown whether the PICL+ firmware limits what values it will accept.
> Thermoelectric cooling drives a real thermal load, and mis-driving one can in
> general cause thermal or condensation damage, so this is the one command
> group where sending arbitrary or swept values carries a plausible
> hardware-damage risk. Until the semantics are understood, an implementation
> should **replay the OEM's exact TEC values in the OEM's sequence, and not
> probe or sweep them.**

## Motor controller (PICM, address 0x24 / 0x44)

| Cmd | Type | Payload | Name | Notes |
|---|---|---|---|---|
| `0x00` | CMD | 0 | ResetMotor | reset controller to initial state |
| `0x82` | WRITE | 3 | FPGA bank `0x82` (TLB) | CCD timing and control: reg + u16 LE. Reg 0 bit 0 is **acquire** |
| `0x84` | WRITE | 3 | FPGA bank `0x84` (TLB) | CCD analog front end: A/D gains and offsets |
| `0x97` | WRITE | 1 | InitMotor | initialise motor controller |
| `0xA0` | CMD | 0 | MotorForward (TLB) | start pulling film |
| `0xA1` | CMD | 0 | MotorReverse (TLB) | start in **reverse**. Not a stop |
| `0xA2` | CMD | 0 | DisengageFilmDrive | release the drive; stops the motor only with acquire off |
| `0xA5` | WRITE | 2 | SetMotorSpeed (TLB) | u16 LE; F-135 400–9500, F-135+ 1000–32766 |

[DOCUMENTED] from the trace. The SCN controller carries both the film motor
and the CCD's FPGA register banks. Note `0x82` and `0x84` collide with light
controller commands; the address disambiguates them.

## Host / bridge (address 0x10)

| Cmd | Type | Payload | Name |
|---|---|---|---|
| `0x84` | WRITE | 1 | With `02`: first half of ResetFifos, before LOW `0x8A` (TLB) |
| `0x85` | CMD | 0 | HostReset: reset the bridge (sent in clusters) |
| `0x8F` | WRITE | 1 | HostSetMode: set bridge operating mode |

Host `0x84` always precedes LOW `0x8A`: together they reset the FIFOs. Not a
per-line handshake. (TLB)

## Sequences

### Initialisation
Bridge reset (HostReset ×3 + HostSetMode) → motor controller init → module-info
read from each controller → FIFO reset → temperature bands → TEC (the Plus) →
LEDs enabled, zeroed, off → resolution flag → FPGA defaults (banks `0x82`,
`0x84`) → disengage → front-panel LEDs → idle poll loop. [DOCUMENTED] as a
fixed sequence, byte-identical across sessions in the trace; step names (TLB).

### Idle / film detection
A tight status-poll loop; periodically the full 30-byte ReadSensorData is
issued. Film insertion changes the sensor values and triggers the scan
workflow. [DOCUMENTED]

### Scan
Calibration if due → FIFO reset → FPGA settings (pixel window, integration,
gains, offsets) → LEDs on at the scan duties → motor speed (`0xA5`) → forward
(`0xA0`) → acquire on (bank `0x82` reg 0 bit 0) → scan-line trigger (LOW
`0x91`). Then **no per-line commands**: the host reads the bulk image endpoint
continuously until it decides the roll has ended (see
[film-transport.md](film-transport.md)). [CONFIRMED in `TLB.dll` and on
hardware]

### End / teardown
Acquire off (bank `0x82` reg 0 bit 0 clear) → LEDs off → FIFO reset →
EndAcquisition (`0x92`) → 20 ms → DisengageFilmDrive (`0xA2`). [CONFIRMED in
`TLB.dll` and seven captures, both models]

Never reset the FIFOs mid-scan: it wedges the bus until a power cycle.
[CONFIRMED on an F-135]

## Open questions

- Names not marked (TLB) are working labels, not OEM identifiers.
- What `0x87` and SCN `0x00` do.
- Payload field layouts are unknown for most write commands.
- The 30-byte sensor response layout is known (position, count, 5-byte
  entries; see the DX page); the meaning of entry types 1, 2, 4 and 6 is not.
- Per-command reply payloads are largely unobserved.
