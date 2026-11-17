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

Rows marked **(TLB)** take their names and payload layouts from the OEM
engine, `TLB.dll` 3.1.0.28 (sha256 5866ec56…). They were read in open-pakon
(https://github.com/jorshhh/open-pakon) in November 2026. These names are
[DOCUMENTED]. A row says [CONFIRMED] only where hardware or a capture shows
the behaviour. [PakonX](https://github.com/amstrd-cpc/PakonX) reached the same
names independently from `TLB.dll` and a different F-135+. The
[changelog](CHANGELOG.md) gives the old labels and why they were wrong.

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
| `0x8A` | CMD | 0 | ResetFifos (TLB) | second half of a FIFO reset, after host `0x84` = `02`. Do not send it during the film pass (see the teardown warning) |
| `0x8B`/`0x8D` | WRITE | 4 | Board temperature bands (TLB) | warning / fault; two s16 LE words, low then high, 0.0625 °C |
| `0x8F`/`0x8C` | WRITE | 4 | Lamp temperature bands (TLB) | warning / fault, relative to the setpoint; same encoding |
| `0x90` | READ | 30 | ReadSensorData | DX/position entries; layout in [dx-barcode.md](dx-barcode.md#how-the-host-reads-it). Read only after the controller raises its service flag [CONFIRMED] |
| `0x91` | WRITE | 3 | DxStart (TLB) | starts the DX reader; the value names the resolution and IR mode. Resets the position counter reported by `0x90` [CONFIRMED]. It does not start the image stream [CONFIRMED in captures] |
| `0x92` | CMD | 0 | DxStop (TLB) | stops the DX reader; closes the DX decode window [CONFIRMED] |
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
| `0x00` | CMD | 0 | FindController (TLB) | presence probe. The reply `07 02 <addr> 00` (status 0) means the controller is present. The OEM sends it twice to `0x44` at every start, and the reply is always `07 02 44 00` [CONFIRMED in captures, F-135+] |
| `0x82` | WRITE | 3 | FPGA bank `0x82` (TLB) | CCD timing and control: reg + u16 LE. Reg 0 bit 0 is **acquire** |
| `0x84` | WRITE | 3 | FPGA bank `0x84` (TLB) | CCD analog front end: A/D gains and offsets |
| `0x97` | WRITE | 1 | InitMotor | initialise motor controller |
| `0xA0` | CMD | 0 | MotorForward (TLB) | start pulling film |
| `0xA1` | CMD | 0 | MotorReverse (TLB) | starts the motor in **reverse**. This command does not stop the motor |
| `0xA2` | CMD | 0 | MotorStop (TLB) | stops the motor. With acquire on, the motor keeps running [CONFIRMED on hardware] |
| `0xA5` | WRITE | 2 | SetMotorSpeed (TLB) | u16 LE; F-135 400–9500, F-135+ 1000–32766 |

> **Warning: `0xA1` is not a stop.** Earlier versions of this page named
> `0xA1` StopFilmDrive. The command starts the motor in reverse. To stop the
> motor, clear the acquire bit and send `0xA2`.

[DOCUMENTED] from the trace. The SCN controller carries both the film motor
and the CCD's FPGA register banks. SCN `0x82` and `0x84` have the same bytes
as two light controller commands. The address tells them apart.

## Host / bridge (address 0x10)

| Cmd | Type | Payload | Name |
|---|---|---|---|
| `0x84` | WRITE | 1 | With `02`: first half of ResetFifos, before LOW `0x8A` (TLB) |
| `0x85` | CMD | 0 | HostReset: reset the bridge (sent in clusters) |
| `0x8F` | WRITE | 1 | HostSetMode: set bridge operating mode |

Host `0x84` always comes before LOW `0x8A`. Together they reset the FIFOs.
The pair is not a per-line handshake. [DOCUMENTED] (TLB)

## Sequences

### Initialisation

The OEM software sends a fixed sequence:

1. Bridge reset: HostReset ×3, then HostSetMode.
2. Motor controller init.
3. Module-info read from each controller.
4. FIFO reset.
5. Temperature bands.
6. TEC (F-135+ only).
7. LEDs enabled, set to zero, then off.
8. Resolution flag.
9. FPGA defaults (banks `0x82` and `0x84`).
10. MotorStop (`0xA2`).
11. Front-panel LEDs.
12. Idle poll loop.

[DOCUMENTED] as a fixed sequence, byte-identical across sessions in the trace.
The step names are from `TLB.dll`.

### Idle / film detection
A tight status-poll loop; periodically the full 30-byte ReadSensorData is
issued. Film insertion changes the sensor values and triggers the scan
workflow. [DOCUMENTED]

### Scan

The OEM software starts a scan in this order:

1. Calibration, if it is due.
2. FIFO reset.
3. FPGA settings: pixel window, integration, gains, offsets.
4. LEDs on at the scan duties.
5. Motor speed (`0xA5`).
6. MotorForward (`0xA0`).
7. Acquire on: bank `0x82` reg 0 bit 0 set.
8. DxStart (LOW `0x91`).

[DOCUMENTED] in `TLB.dll`. Setting acquire starts the image stream, and
`0x91` does not. [CONFIRMED in captures, F-135+ serial 16402] In every
session, image data starts 3 to 6 ms after the OEM sets acquire. The OEM also
sends `0x91` once during setup, with acquire off. No image data follows it
until acquire is set, 59 ms to 31 s later.

After step 8 the host sends no per-line commands. It reads the bulk image
endpoint continuously until it decides that the roll has ended (see
[film-transport.md](film-transport.md)). [CONFIRMED on hardware]

### End / teardown

> **Warning: do not reset the FIFOs during the film pass.** On an F-135, a
> FIFO reset (host `0x84`, LOW `0x8A`) sent while film was moving stopped the
> bus. The bus did not respond until the scanner was turned off and on.
> [CONFIRMED on an F-135, May 2026, open-pakon] The OEM resets the FIFOs 128
> times during calibration, with acquire on and the motor stopped, and never
> during the film pass. [CONFIRMED in captures, F-135+]

The OEM software ends a scan in this order:

1. Acquire off: bank `0x82` reg 0 bit 0 clear.
2. LEDs off.
3. FIFO reset.
4. DxStop (`0x92`).
5. Wait at least 20 ms.
6. MotorStop (`0xA2`).

[DOCUMENTED] in `TLB.dll`. [CONFIRMED in captures, both models] Image data
stops about 1 ms after acquire is cleared. `0x92` comes 31 to 51 ms later, and
`0xA2` follows `0x92` by 23 to 31 ms.

## Open questions

- Names not marked (TLB) are working labels, not OEM identifiers.
- What LOW `0x87` does.
- Payload field layouts are unknown for most write commands.
- The 30-byte sensor response layout is known (position, count, 5-byte
  entries; see the DX page); the meaning of entry types 1, 2, 4 and 6 is not.
- Per-command reply payloads are largely unobserved.
