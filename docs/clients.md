# Open-source clients compared

## Overview

| Client | Published | Approach | Platforms | Models | License |
|---|---|---|---|---|---|
| [PakonClient](https://github.com/eatfrog/PakonClient) | 2022-05-28 | Drives the OEM COM server (`tlx.dll`) on Windows. | Windows | F-135+ | none stated |
| [psix](https://github.com/veroc/psix) | 2026-06-30 | Native driver and a local web app. | Linux | F-135+ | LGPL-3.0 |
| [open-pakon](https://github.com/jorshhh/open-pakon) | 2026-08-12 | Native app in Rust. Uses Kodak's colour engine for colour. | macOS, Windows, Linux | F-135, F-135+ | AGPL-3.0 |
| [pakon-tlx-macos](https://github.com/pablonavarrob/pakon-tlx-macos) | 2026-08-17 | Runs the unmodified OEM software under Wine, with a libusb shim in place of the Windows driver. | macOS | F-135, F-135+ | MIT |
| [pakon-mac](https://github.com/gazzdingo/pakon-mac) | 2026-10-24 | Driver library and command line in C99. | macOS (libusb) | F-135+; F-235 and F-335 in beta | none chosen; all rights reserved |
| [PakonX](https://github.com/amstrd-cpc/PakonX) | 2026-11-17 | Native driver in C++ and a desktop app. | Windows; macOS and Linux (loader not yet run on hardware) | F-135+ (an F-135 is detected, then refused) | GPL-3.0 |

## Firmware, colour and per-unit data

The scanner needs Kodak's USB firmware at every power-on (see
[USB identity & firmware](usb-identity-and-firmware.md)). The EEPROM column
matters most: the per-unit EEPROM holds factory calibration that exists
nowhere else, and a client that writes it can damage the scanner. Read
[Per-unit data & safety](per-unit-data-and-safety.md) before you use a
client that writes it.

| Client | Firmware comes from | Colour | Per-unit EEPROM | Checked |
|---|---|---|---|---|
| PakonClient | the user's own OEM install | the OEM software | the OEM software | 2026-11-17, README; 2026-08-17, code |
| psix | firmware files the user supplies | its own pipeline and dust removal | not read, apart from the boot personality record | 2026-11-17, README; 2026-08-17, code |
| open-pakon | Kodak firmware committed in its own repository | Kodak's colour engine (`PakonIMAu.dll`) and scene-balance data, committed in its repository; own dust removal | reads it and saves a copy | 2026-11-17, README and file list |
| pakon-tlx-macos | the OEM install, which its setup script fetches from a third-party mirror onto the user's machine | the OEM software | OEM software; plus a read-only dump tool | 2026-11-17, README; 2026-08-17, code |
| pakon-mac | the user's own copy of the PSI 3.0.4.27 install, checked by SHA-256 | reads tables taken from the user's PSI install; not yet available in the public tree | reads it once per scanner and stores the copy; no write path found | 2026-11-17, README and code |
| PakonX | Kodak firmware bundled in its installers, downloaded at build time from a third-party mirror | its own pipeline, or on Windows Kodak's colour engine, bundled in the Windows installer | reads it; writes are blocked in code | 2026-11-17, code |

Checked gives the date of the check and what was read: the project's
README, its code, or both. A cell taken only from a README is that project's
claim, not a fact checked here.
