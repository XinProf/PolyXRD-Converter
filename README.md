# PolyXRD Converter

> [Chinese (Simplified)](README-CN.md) · English

**XRD spectral format converter**  —  a lightweight tool spun off from [PolyXRD](https://github.com/PolyXRD/PolyXRD): it converts pattern files exported by various vendors/software into the target format you need, with **automatic recognition of multi-vendor binary RAW**; just double-click to run, no installation required.

**Current version v1.5.1** | [Download the latest Release](https://github.com/PolyXRD/PolyXRD-Converter/releases/latest) (Setup installer / Internal single-file exe, with SHA-256) | UI languages: Simplified Chinese / English / Japanese

<div align="center"><img src="src/polyxrd_converter/resources/icon.png" alt="PolyXRD Converter" width="96"></div>

## Features

- **Read**: `xy` `txt` `dat` `csv` `smz` `mdi` `xrdml/xml` `brml` `json`, GSAS powder data (`gsas`/`fxye`/`gda`, with four BANK types FXYE/FXY/ESD/STD), FullProf `.dat`, Rigaku `.ras`/`.rasx`, Topas `.xye`/`.qye`, Fit2D `.chi`/`.qchi`, and multi-vendor binary `RAW` (auto-detected, see the table below); Shimadzu project files `.ORG/.RSM/.RBG/.RK1/.RAX/.RLZ/.RFI` share the same structure as `.RAW` and are supported as well
- **Write**: `xy` `txt` `dat` `csv` `mdi` `raw` `xrdml` `json`
- **Batch conversion**: drag in an entire directory; a background thread converts files one by one with live progress
- **Pattern preview**: hand-drawn curves (no matplotlib dependency), wheel zoom / drag pan / double-click reset / strongest-peak marker
- **Angle-axis guardrail**: when RAW header parameters are corrupted, it **flags them as untrustworthy rather than guessing** — automatic conversion is refused and the user is guided to specify `(start 2θ, step)` manually, preventing patterns with wrong peak positions from being produced
- **Trilingual UI**: Simplified Chinese / English / Japanese; switching via the "Language" menu takes **effect immediately** (no restart); language names are written in their native scripts, and adding a new language only requires adding a table in `i18n.py`
- **Dual build forms (v1.5.0)**: two binaries — the Setup installation edition (with registration) and the Internal evaluation edition (without registration); see "Two editions" below

## Two editions (since v1.5.0)

| | **Setup installation edition** | **Internal evaluation edition** |
|---|---|---|
| Artifact | `PolyXRD-Converter-Setup-vX.Y.Z.exe` (Inno Setup installer) | `PolyXRD-Converter-Internal-vX.Y.Z.exe` (single file, double-click to run) |
| Registration | ✅ Yes (see "Registration" below) | ❌ No |
| Status bar | Unregistered / Registered: organization/department name (right-aligned) | Fixed "Internal Evaluation Version" (centered, bold, red text across the whole status bar) |
| Use case | External distribution / copyright management | Internal evaluation use |

## Registration (Setup installation edition)

- **Unregistered**: a prompt appears on every launch — "This software is unregistered. All rights reserved. Commercial use is prohibited. To register, please contact sshztx@outlook.com" — click "OK" to keep using it normally (prompt only); click "Enter Registration Code" to register on the spot
- **Obtaining a registration code**: send your **organization/department name** to sshztx@outlook.com; the developer generates a code of the form `PXRD1-XXXX-XXXX-...` (HMAC-SHA256 offline algorithm, shared across all PolyXRD family software)
- **Entering a registration code**: menu bar "Help" → "Enter Registration Code (&R)…", just paste it; accepts codes with/without prefix, arbitrary spaces and hyphens, and mixed case
- **After registration**: the **far right** of the status bar shows "Registered: organization/department name"; no prompt on subsequent launches; the same code can be used on multiple computers within the same organization (not machine-bound)
- **Registration state**: stored in `%APPDATA%\PolyXRD\Registration\PolyXRD_Converter.json`; the code is re-verified on every launch (hand-editing the JSON to forge it is ineffective); uninstalling does not clear it, so it remains registered after reinstallation

## Multi-vendor RAW auto-detection

| Vendor / format | Magic number | Detection notes |
|---|---|---|
| Shimadzu XRD | `Shimadzu XRD` | Proprietary container: variable-length header + **int32 LE** intensities. Point count taken from `u32@0x34 ÷ 4` in the header (authoritative); data start = `u32@0x30 + 56` (header length varies; some files count the embedded source path into the header). Start/end/step are `i32@0x12C/0x130/0x134 ×1e-4` (**signed**, so negative-angle θ scans are supported). The `0x20` record type distinguishes `.RAW/.ORG/.RSM/.RBG/.RK1/.RAX/.RLZ/.RFI` (same structure, all readable) from `PKR/PKK/PKS/PSE/PKF` (peak tables, **explicitly rejected**). **Truncation guard**: when the declared data region extends past the end of the file, it is clamped to the actual usable point count, marked `truncated`, and highlighted — header bytes are never read as intensities; the angle axis of truncated files is forced to manual specification |
| Rigaku (RINT/SmartLab) | `FI\x00\x00` | Fixed 3162-byte header + variable-length float32 intensities, axis parameters at 0xB92; point-count consistency check using two pieces of evidence (n_axis vs n_fixed) |
| Scintag RAW2 | `RAW2` | 316-byte header; point count (int16)@0x102, step (float32)@0x10C, start 2θ (float32)@0x110 |
| Bruker/Siemens RAW v1 | `RAW1.01` / `RAW1.00` / `RAW ` | Fixed-length string header + 10 × 892-byte range records + trailing float32 intensities; the axis is matched against strict criteria (step 0.005~0.5°, span≥1°, end≤180°), and is flagged as untrustworthy if no match |
| MDI / Jade text export | — (two-line header) | Header line `start step mode anode λ stop npoints`, single-column fixed-width intensities; the axis is built from the header only after cross-checking the target-material symbol against the wavelength |
| Unknown RAW | — | Fallback parsing (longest plausible float32 suffix); the axis is always flagged as untrustworthy |

> **Design principle**: the parser is organized around "magic-number routing + confidence flags". When header parameters are corrupted (e.g. end > 180°, or an inconsistent point count), it produces `axis_confident=False`, the GUI highlights a warning, and conversion is blocked until the user supplies the angle axis manually.

### Four conventions of the Shimadzu container (all handled)

- **Truncated step field**: `Std/Search.RAW` writes 0.0213 in its header, but the actual value is 0.0212644 (truncated to 4 decimal places). When the point-count consistency check fails, the step is back-computed as `(end − start)/(n − 1)`.
- **The header end is off by one step**: some samples use `end = start + step×n` (e.g. `Stress/Heikei-Test`: 34.0 + 0.05×80 = 38.0, closing exactly), while others use `start + step×(n−1)`. The point count declared by `0x34` and the step field in the header are authoritative, and both conventions are accepted — uniformly rebuilding the axis as `(end−start)/(n−1)` would, for these files, stretch the axis into a step the firmware could never produce (e.g. 0.05063291).
- **The wavelength field may be stale**: in `Stress/std-sokkei.RAW` the target material is already Cr, but the wavelength field still holds Cu 1.5406 — the target-element symbol lookup table is authoritative.
- **Truncated files**: when the data region declared in the header does not fit the actual file (incomplete transfer, etc.), it is clamped to the usable point count, marked `truncated`, and highlighted as "File incomplete" in the list; for truncated files `end` corresponds to the declared point count rather than the actual one, so the angle axis is untrustworthy and must be specified manually before conversion.

## Usage

### Graphical user interface

Double-click `PolyXRD-Converter.exe` (or run `python main.py`):

1. Drag files/folders into the window (or click "Add Files")
2. The table automatically detects each file's format, point count, 2θ range, and strongest-peak intensity
3. Select any row to preview the pattern on the right
4. Choose the target format + output directory → "Start Conversion"
5. Switch the UI language: menu bar "Language" → Simplified Chinese / English / Japanese, **effective immediately**, no restart needed
6. View the license: menu bar "Help" → "Copyright" (alongside "About") opens a bilingual Chinese-English license notice and contact information

Launching with arguments (equivalent to dragging files onto the exe icon):

```
PolyXRD-Converter.exe D:\data\spectrum.raw D:\data\another.xy
```

### Command line / as a library

```python
from polyxrd_converter.loader import load_pattern
from polyxrd_converter.converter import convert_file

data = load_pattern("spectrum.raw")          # format auto-detection
print(len(data), data.two_theta[0], data.two_theta[-1])

convert_file("spectrum.raw", "spectrum.xrdml")   # single-file conversion
```

## Validation

Full cross-check on real samples (point-by-point comparison against exports of the same pattern in other formats):

```
python tests/validate_samples.py     # 612 assertions
python tests/smoke_gui.py            # 65 offscreen GUI smoke tests
python tests/test_registration.py    # 21 registration checks (pure stdlib)
```

Coverage:
- **Shimadzu**: all 72 `.RAW` files in `XRData/Shimadzu Data` parsed successfully (all angle axes trustworthy); 5 sets were compared point-by-point with the vendor's official ASCII exports  —  **intensities exactly equal**, 2θ difference <3e-14; includes 5 boundary-sample regressions (negative-angle θ scans, truncated step, end off by one step, Cr target, variable-length header) plus peak-table rejection and synthetic truncated-file regressions
- **Full Shimadzu cross-check (v1.1.0)**: all 305 Shimadzu container files in the `XRData` library (including the 1608/976-byte variable-length-header subtypes) parsed successfully, with 26 genuinely truncated files correctly flagged and blocked; 206 files were compared point-by-point against the PolyXRD main program's `DataLoader._load_shimadzu_raw`, **intensities all equal**; 13 `.RAW`↔PCXRD ASCII export pairs for `smz geshi` were compared point-by-point (including single-axis θ rocking scans), intensities all equal, 2θ difference <1.5e-14
- **Rigaku**: 39 Rigaku FI files (point-by-point identical to `.txt`)
- **Scintag**: 33 RAW2 files (point-by-point identical to `.dat/.txt/.xy`)
- **Bruker**: the RAW v1 sample with a corrupted axis is correctly blocked
- **Jade**: text exports build the axis from the header + peak-position cross-check
- **GSAS-II family text formats**: `.gsas/.fxye` (FXYE/FXY/ESD/STD), FullProf `.dat`, Rigaku `.ras/.rasx`, Topas `.xye/.qye`, Fit2D `.chi/.qchi` — full-format regression on synthetic samples, plus point-by-point comparison against GSAS-II itself (Argonne's open-source reference implementation) as the oracle  —  10 formats fully agree with GSAS-II; the 4 skips are all GSAS-II's own defects (STD `int()` crash, Fit2D header falling into the pyFAI branch, etc.), all of which this implementation avoids
- Round-trip conservation for 8 formats
- **Trilingual consistency (v1.5.0)**: 141 strings × 3 languages have aligned key sets, placeholders with the same names and counts, and every string passes `format`; read/write format names are translated in all three languages; 7 strings such as buttons/menus/"About" are guarded by a "all three languages differ" check (to catch whole-table copies left unchanged). Another 3 UI smoke assertions: after switching to ja/en/zh, buttons, table headers, the menu bar, the target-format dropdown, and the "Detected Format" column of loaded rows are all re-translated immediately
- **Registration mechanism (v1.5.0)**: 21 code-generation/verification checks (round-trip of Chinese/English organization names, input tolerance, tamper rejection, state-file anti-forgery, trilingual status strings); 12 GUI smoke checks (unregistered-prompt flow, status bar showing the organization name after registration and re-translating immediately with the language, internal evaluation edition having no registration menu and showing "Internal Evaluation Version")
- **Internal evaluation edition status bar (v1.5.1)**: 3 new smoke checks  —  the prompt text is horizontally centered (`AlignHCenter`), the font is bold, and the style is red `#ff0000`; also measured under native Windows rendering at three window widths 900 / 1100 / 1600, the offset between the actual rendered center of the red text and the geometric center of the status bar is consistently **+0.5px** (the width QStatusBar reserves for the bottom-right size grip is compensated by the label's left margin; the reserved width is back-derived from the geometric difference, adapting for both offscreen 24px and native 20px); boldness verified by A/B rendering (red pixels 209 → 338 at the same font size), with the font size still at the global 9.5pt, unchanged

## Documentation

- [Project handover record](docs/交接记录-HANDOVER.md) — project positioning, module layering, RAW parser registry, acceptance matrix and ground-truth sources, version progress, 24 lessons learned, known issues (must-read for incoming developers / collaboration model)

## Development

```bash
python -m venv venv
venv\Scripts\pip install -e . pyinstaller
python main.py                 # run from source
build.bat                      # dual-build packaging: Internal single-file + Setup installer (requires Inno Setup 6)
```

### Adding a new language

In `src/polyxrd_converter/i18n.py`, add a table with the same keys as `ZH` and hook it into `_LANGS` and `LANGS`; the language menu is then generated automatically.
Key-set and placeholder consistency is enforced by `tests/validate_samples.py::validate_i18n` (when a key is missing, `tr()` silently falls back to Chinese).

## Dependencies

Runtime requires only `numpy` + `PySide6` (the preview is hand-drawn with QPainter; matplotlib/pyqtgraph are deliberately avoided to keep the single-file size and startup speed under control).

## License

Copyright (c) 2026 PolyXRD Team.

This software may be used free of charge for academic research and teaching.
Commercial use is prohibited without separate written permission from the copyright holder.
The software is provided as-is, without any warranty.

See the full terms in [`LICENSE`](LICENSE) (English) and [`LICENSE-CN`](LICENSE-CN) (Chinese); for commercial licensing, contact: sshztx@outlook.com.
When using this software, also comply with the upstream third-party license terms of PySide6 (LGPL/GPL), numpy (BSD), etc.

---

<div align="right">
  <i>PolyXRD Team · 2025 — 2026 · Contact: sshztx@outlook.com</i>
</div>
