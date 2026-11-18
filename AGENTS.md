# Instructions for agents

This repository is a reference for how the Kodak/Pakon F-X35 film scanners
work. It is published as a site from `docs/` with MkDocs. Pull requests from
people and from agents are welcome. This file tells an agent what a pull
request must do before a person reviews it.

Read [docs/CONVENTIONS.md](docs/CONVENTIONS.md) before you change a page.
Every rule below comes from it.

## What a change may contain

- Facts about the hardware, the protocol and the processing. See "Scope" in
  the conventions.
- No OEM software, firmware bytes, OEM data files, decompiler output or
  scanned documents. Give the fact you derived, not the file you derived it
  from.

## How to write a fact

- Give every fact that is not obvious a confidence marker: `[CONFIRMED]`,
  `[DOCUMENTED]`, `[INFERRED]` or `[SPECULATIVE]`. The marker must match the
  evidence. A behaviour read from a DLL, and not seen on hardware or in a
  capture, is not `[CONFIRMED]`.
- Say how the fact is known. If the source is an OEM DLL, give the file name,
  its version and the first characters of its SHA-256.
- Say which actor does the action: the scanner, or the OEM software. If a fact
  is about the OEM software, put it in the "A note about the Pakon
  implementation" callout.
- Take dates from a record. Do not estimate them.
- Follow the "Writing" section of the conventions: short sentences, one fact
  in each, full sentences, no em dashes.
- Write only the facts. Do not describe the page, introduce a table with a
  sentence, or explain policy on a fact page. See "What to leave out" in the
  conventions.

## When a fact changes

- Change the page so that it states the current fact. Do not leave a note
  such as "this page used to say" on the page. Keep a note only when the old
  claim was unsafe to act on.
- Add a changelog entry to [docs/CHANGELOG.md](docs/CHANGELOG.md). Say what
  the page said, what it says now, and why the old claim was wrong.
- If you rename a command or a register, change the name on every page in the
  same pull request. Search the whole of `docs/` and the README for the old
  name.

## Before you open the pull request

1. Make a virtual environment in the repository and install MkDocs Material
   into it. The `.venv/` directory is ignored by git.

   ```
   python3 -m venv .venv
   .venv/bin/pip install mkdocs-material
   ```

   Use a current Python 3. An old `pip` (such as 20.x) cannot install the
   dependencies, and fails with an `IndexError` in its TOML parser.
2. Build the site: `.venv/bin/mkdocs build --strict`. Fix every warning.
3. Search `docs/` for text your change makes wrong, and correct it too.
