# Open-source clients compared

This page compares the open-source clients for the F-135 and F-135+. It
says what each client does and where it gets the files it needs. It does not
rate them. Use it to find the client that fits your scanner and your
computer, and to find code that implements a fact in this reference.

Clients change often. Each row gives the date it was checked and what was
read: the project's README, its code, or both. A cell taken only from a
README is a claim by that project, not a fact checked here. If a row is out
of date, [open an issue](https://github.com/alibosworth/pakon-reference/issues).

## What each client is

| Client | Approach | Platforms | Models | License |
|---|---|---|---|---|
| [open-pakon](https://github.com/jorshhh/open-pakon) | Native app in Rust. Uses Kodak's colour engine for colour. | macOS, Windows, Linux | F-135, F-135+ | AGPL-3.0 |
| [pakon-mac](https://github.com/gazzdingo/pakon-mac) | Driver library and command line in C99. | macOS (libusb) | F-135+; F-235 and F-335 in beta | none chosen; all rights reserved |
| [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos) | Runs the unmodified OEM software under Wine, with a libusb shim in place of the Windows driver. | macOS | F-135, F-135+ | MIT |
| [PakonClient](https://github.com/eatfrog/PakonClient) | Drives the OEM COM server (`tlx.dll`) on Windows. | Windows | F-135+ | none stated |
| [PakonX](https://github.com/amstrd-cpc/PakonX) | Native driver in C++ and a desktop app. | Windows; macOS and Linux (loader not yet run on hardware) | F-135+ (an F-135 is detected, then refused) | GPL-3.0 |
| [psix](https://github.com/veroc/psix) | Native driver and a local web app. | Linux | F-135+ | LGPL-3.0 |

## Firmware, colour and per-unit data

The scanner needs Kodak's USB firmware at every power-on (see
[USB identity & firmware](usb-identity-and-firmware.md)). The EEPROM column
matters most: the per-unit EEPROM holds factory calibration that exists
nowhere else, and a client that writes it can damage the scanner. Read
[Per-unit data & safety](per-unit-data-and-safety.md) before you use a
client that writes it.

| Client | Firmware comes from | Colour | Per-unit EEPROM | Checked |
|---|---|---|---|---|
| open-pakon | Kodak firmware committed in its own repository | Kodak's colour engine (`PakonIMAu.dll`) and scene-balance data, committed in its repository; own dust removal | reads it and saves a copy | 2026-11-17, README and file list |
| pakon-mac | the user's own copy of the PSI 3.0.4.27 install, checked by SHA-256 | reads tables taken from the user's PSI install; not yet available in the public tree | reads it once per scanner and stores the copy; no write path found | 2026-11-17, README and code |
| pakon-tlx-macos | the OEM install, which its setup script fetches from a third-party mirror onto the user's machine | the OEM software | OEM software; plus a read-only dump tool | 2026-11-17, README; 2026-08-17, code |
| PakonClient | the user's own OEM install | the OEM software | the OEM software | 2026-11-17, README; 2026-08-17, code |
| PakonX | Kodak firmware bundled in its installers, downloaded at build time from a third-party mirror | its own pipeline, or on Windows Kodak's colour engine, bundled in the Windows installer | reads it; writes are blocked in code | 2026-11-17, code |
| psix | firmware files the user supplies | its own pipeline and dust removal | not read, apart from the boot personality record | 2026-11-17, README; 2026-08-17, code |

## Not listed

Closed-source clients are not listed. A client is added when its source is
public. To add one, open an issue with a link to it.
