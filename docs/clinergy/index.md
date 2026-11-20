---
layout: default
title: "Clinergy: extractor and running the demo in Wine"
permalink: /docs/clinergy/
---

# Clinergy: extractor and running the demo in Wine

Clinergy was a structured data entry system for primary care built on the PEN&PAD approach, "built on top of a live GALEN terminology server delivering an adaptation of a GALEN Common Reference Model" ([OpenGALEN software page](https://opengalen.eggbird.eu/sources/software.html)). Alan Rector demonstrates it in [interview part 1](/interviews/alan-rector-part-1/#t=685), and the [thumb pain walkthrough](/docs/clinergy-thumb-pain-demo/) goes through that demo screen by screen. This page covers the YAML extractor and how to run the 1998 Windows demo under Wine on Linux. See also [front page](/#clinergy).

## Demo download

[ClinergyDemo.zip](https://opengalen.eggbird.eu/download/ClinergyDemo.zip) (6 MB, a 1998 Windows installer) is on the OpenGALEN mirror. There is also a [copy archived from opengalen.org](https://web.archive.org/web/20070818150258/http://www.opengalen.org/download/clinergydemo.zip) on the Internet Archive. Its raw capture (`…/20070818150258id_/…`) is byte-identical to the mirror file (sha256 `f138c38b24de630ce0c89bc8018dbc68fc2b17e040235c9c01e10761d8114270`, checked 2026-11-18). The demo is © 1997-8 Semantic Technologies Ltd / Victoria University of Manchester, All Rights Reserved.

## Extractor
[clinergy-extractor](https://github.com/erik-sundvalls-bots/clinergy-extractor) turns your copy of the zip into human-readable YAML: one file per data-entry form, plus concepts, examinations, summary-sentence rules and body maps. 

A list of the forms is [available here on the website](https://github.com/pen-pad-revival/pen-pad-revival.github.io/blob/main/docs/clinergy/list-of-clinergy-forms.md)

The extractor also helps creating a first vereion of map between Clinergy names to OpenGALEN 8: exact matches first, then a review worklist for a person or an AI agent, using a small OpenGALEN MCP server.

It also exports the body-map pictures, icons, cursors and bitmaps to `images/` (BMP/ICO/CUR converted to PNG; ICO/CUR originals kept; WMF kept as `.wmf`), with an `images.yaml` index, and writes body-map hotspot coordinates in `bodymap.yaml`.

It needs [Deno](https://deno.com/), Wine and the 32-bit MinGW compiler (details in the extractor [README](https://github.com/erik-sundvalls-bots/clinergy-extractor)):

```sh
git clone https://github.com/erik-sundvalls-bots/clinergy-extractor && cd clinergy-extractor
deno task extract ClinergyDemo.zip out/
```

The output is a private study copy. Please don't publish it. The repository contains only the extractor code, not the demo or any extracted Clinergy content.

## Running Clinergy in Wine on Linux

The demo itself runs under Wine 10 (tested on Debian 13). The extractor repo has a script for the basic install:

* Use a 32-bit prefix (`WINEARCH=win32`) set to Windows XP. On Debian this needs the i386 architecture: `sudo dpkg --add-architecture i386 && sudo apt update && sudo apt install wine wine32:i386 gcc-mingw-w64-i686`.
* Skip the 16-bit `SETUP.EXE` and unpack the installer payload yourself. The script uses the extractor's unpacker to put the programs in `C:\Clinergy\Programs`, the model, INI and bitmap files in `C:\Clinergy\Compdata`, and `Penpad.ini` in `C:\windows`.
* Use the bundled `Mfc42.dll`, but keep Wine's own `msvcrt` instead of the 1997 `Msvcrt.dll`: `WINEDLLOVERRIDES="mscoree,mshtml,winemenubuilder.exe=d;mfc42,mfc42u=n;msvcrt=b"`.
* Start `penpad.exe` from `C:\Clinergy\Programs`, then load the model by the name `model`.

```sh
deno run -A clinergy-wine.ts install ClinergyDemo.zip   # in the clinergy-extractor folder
deno run -A clinergy-wine.ts demo                       # Pen&Pad with the model loaded and the Symptom Selector open
```

### The fuller setup: menu bar and host demos

[`scripts/setup-wine-clinergy.sh`](https://github.com/pen-pad-revival/pen-pad-revival.github.io/blob/main/scripts/setup-wine-clinergy.sh) in this site's repository runs the install above and then adds what is needed for the rest of the demo. It downloads `ClinergyDemo.zip` (checked against a sha256) and does not contain any Clinergy files. By default it uses the mirror. `SOURCE=archive` (or `--source=archive` as the first argument) uses the Internet Archive capture instead, and `SOURCE=auto` tries the mirror first and falls back to the archive. The VB4 runtime and controls it registers come from the same zip, so nothing else is downloaded except the clinergy-extractor checkout from GitHub.

```sh
scripts/setup-wine-clinergy.sh install    # needs wine (32-bit), i686-w64-mingw32-gcc, deno, git, curl
scripts/setup-wine-clinergy.sh penpad     # Pen&Pad plus the ppmenu menu bar
scripts/setup-wine-clinergy.sh windemo    # or vbdemo, gendemo
scripts/setup-wine-clinergy.sh kill
SOURCE=archive scripts/setup-wine-clinergy.sh fetch   # only download and verify, here from the Internet Archive
```

Without a `DISPLAY` it starts its own Xvfb on `:78`. What it adds:

* **A Wine virtual desktop** (`HKCU\Software\Wine\Explorer`, 1260×880). Without a window manager, popup menus and title bars did not work reliably.
* **ppmenu, a stand-in menu bar.** Pen&Pad's frame window shows no menu bar under Wine, even in the virtual desktop. [`scripts/clinergy/ppmenu.c`](https://github.com/pen-pad-revival/pen-pad-revival.github.io/blob/main/scripts/clinergy/ppmenu.c) (our own code, about 60 lines) loads the original menu resource 129 from `penpad.exe` as a data file and shows it on a small always-on-top window at the bottom of the screen. It forwards each command to the Pen&Pad frame as the same `WM_COMMAND` the real menu sends. That makes every menu item reachable, for example Model > Load model and DEM > Symptom, Disease or Common Topics.
* **The VB4 runtime and controls for the host demos.** The 16-bit installer would have registered these. The script copies `VB40032.DLL`, `MFC40.DLL`, `MSVCRT40.DLL`, `GRDKRN32.DLL`, `gswdll32.dll`, `msvcrt20.dll` and the `*.OCX` files from `C:\Clinergy\Programs` into `system32` and runs `regsvr32` on each OCX. It also copies `ven2232.olb` to `C:\windows\system`, the path recorded in the VB programs. Without it, Win Demo stops with "Run-time error 48: Error in loading DLL" when a patient is selected.
* **OLE registration.** It runs `VBDemo.exe /regserver` to register `VBPenPadHost.VBHostConnector`, the class Pen&Pad calls back when a session is accepted. It also runs `penpad.exe /regserver` to register `PENPAD.CONNECT`.

Menu command IDs in `penpad.exe` (menu 129), useful for scripting with `PostMessage`:

| Menu | Commands (ID) |
|---|---|
| Model | Load 32771, Delete 32775, Set User 32778, Term Server 32790 |
| DEM | Launch DEM 32776, Symptom 32772, Disease 32773, Common Topics 32774, Customise 32789 |
| View | Toolbar 59392, Status Bar 59393, Hide Menus 32777 |
| Host Prog | Read Browser 32786, Call Hostess 32788 |

### Status (tested 2026-11-18)

| Part | Under Wine |
|---|---|
| Pen&Pad, model load, Symptom Selector (`demo`) | Works |
| All Pen&Pad menu commands via ppmenu | Work (Load model dialog, "Model loaded OK") |
| VB Demo (`VB Host For Clinergy`) | Starts Pen&Pad over OLE: "Connected to Pen&Pad successfully", "Model loaded OK" |
| Win Demo (problem-oriented record mock-up) | Login, patient list and patient record display work. No `penpad.mdb` is needed: the patients and problems come from the `.prc` files in `Win Demo` |
| Gen Demo (character-based record mock-up) | Starts, shows the record, and launches Pen&Pad |
| Starting a data entry session from any host demo (QDE, SYM, DIS ...) | **Fails.** VB Demo says "Error returned from StartOLEDataEntrySession", Win and Gen Demo say "Error returned from StartDataEntrySession" |

What the Wine trace shows for the failing session start: during `StartOLEDataEntrySession` an RPC stub call fails with an access violation inside `VB40032.DLL` in the host process (`CStdStubBuffer_Invoke ... exception 0xc0000005`). Pen&Pad then gets RPC fault `0x3e6` and returns an error code. The likely cause is Wine's OLE marshalling of the 1996 VB4 out-of-process objects, but this has not been confirmed. Two other things were noticed. First, all three host programs register the same ProgID `VBPenPadHost.VBHostConnector`, so the last one registered wins. Second, re-registering VB Demo just before running it did not fix the error. Data entry through Pen&Pad's own DEM menu works.

On a brand-new prefix the first host demo start can stop with "Run-time error 429: OLE Automation server can't create object". After `kill`, the second start worked.

### Related: the GALEN terminology tools

The [OpenGALEN tools on Linux](/docs/galen-tools/) page covers the ClaW/ROIS terminology server, the GCE browser and OpenKnoME under Wine, with the full OpenGALEN 8 model. It also has a section on what the running programs show that is useful when designing modern web versions.
