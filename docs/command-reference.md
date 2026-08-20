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

See [ppb-protocol.md](ppb-protocol.md) for the frame format these ride in.

## Light / CCD controller (PICL, address 0x20 / 0x40)

| Cmd | Type | Payload | Name | Notes |
|---|---|---|---|---|
| `0x80` | WRITE | 1 | SetCcdConfig | general CCD config; sent repeatedly during init |
| `0x81` | WRITE | 5 | SetCcdGainOffset | CCD gain/offset [INFERRED layout] |
| `0x82` | WRITE | 12 | SetColorMatrix | colour-correction matrix data [INFERRED] |
| `0x83` | READ | 1 | ReadCcdStatus | ready/busy/error flags |
| `0x84` | READ | 2 | ReadLightStatus | lamp level + status |
| `0x87` | WRITE | 2 | SetLightPower | light source power/PWM level |
| `0x88` | READ | 4 | ReadTemperature | temperature sensors |
| `0x89` | WRITE | 1 | EnableScan | arm the CCD for scanning |
| `0x8A` | CMD | 0 | AcquireLine | trigger one CCD line acquisition |
| `0x8B`/`0x8C`/`0x8D` | WRITE | 4 | SetCcdExposure B/G/R | per-channel exposure timing |
| `0x8F` | WRITE | 4 | SetLightConfig | light source configuration |
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
| `0x82` | WRITE | 3 | SetMotorSpeed | speed (16-bit, tenths of mm/s) + mode; indexed by sub-register |
| `0x84` | WRITE | 3 | SetMotorConfig | acceleration/deceleration/step-mode profile |
| `0x97` | WRITE | 1 | InitMotor | initialise motor controller |
| `0xA0` | CMD | 0 | EngageFilmDrive | start pulling film |
| `0xA1` | CMD | 0 | StopFilmDrive | immediate stop |
| `0xA2` | CMD | 0 | DisengageFilmDrive | release the drive |
| `0xA5` | WRITE | 2 | SetMotorCalibration | home/adjust parameters, before engage |

[DOCUMENTED] from the trace. Note `0x82` collides with the light controller's
SetColorMatrix; the address disambiguates them. Advance speed is in tenths of
mm/s; the OEM UI exposes forward 10–355 and reverse −10 to −355, with 0
invalid.

## Host / bridge (address 0x10)

| Cmd | Type | Payload | Name |
|---|---|---|---|
| `0x84` | WRITE | 1 | HostReady: signals readiness for the next scan line |
| `0x85` | CMD | 0 | HostReset: reset the bridge (sent in clusters) |
| `0x8F` | WRITE | 1 | HostSetMode: set bridge operating mode |

[DOCUMENTED] HostReady immediately precedes each AcquireLine; the counts match
exactly in the trace.

## Sequences

### Initialisation
Bridge reset (HostReset ×3 + HostSetMode) → motor reset + init → module-info
read from each controller → CCD/light configuration (light config, per-channel
exposure, TEC on the Plus, colour matrix, enable) → motor speed ramp + config →
disengage to clear the film path → enter an idle poll loop. [DOCUMENTED] as a
fixed sequence, byte-identical across sessions in the trace.

### Idle / film detection
A tight status-poll loop; periodically the full 30-byte ReadSensorData is
issued. Film insertion changes the sensor values and triggers the scan
workflow. [DOCUMENTED]

### Scan
Pre-scan calibration (dark/bright reference lines) → motor engage → a tight
loop of HostReady→AcquireLine pairs with status polls, while image data streams
out the bulk image endpoint → EndAcquisition → DisengageFilmDrive. Exposure and
gain are locked before the scan begins (the gain/matrix commands do not appear
between engage and end). [DOCUMENTED]

### End / teardown
CCD config to disable acquisition → EndAcquisition → DisengageFilmDrive → motor
speed to an exit value → idle. [DOCUMENTED]

## Open questions

- Most command names are working labels, not OEM identifiers.
- Payload field layouts are unknown for most write commands.
- The 30-byte sensor response layout is known (position, count, 5-byte
  entries; see the DX page); the meaning of entry types 1, 2, 4 and 6 is not.
- Per-command reply payloads are largely unobserved.
