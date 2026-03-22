---
name: chip-identification
description: Reference for identified chips/ICs, cards, and GPU carrier systems from PCB photos. Use this skill when identifying or looking up chips, riser cards, or GPU baseboards found in the user's hardware collection.
---
# Chip Identification Reference

A log of chips identified from macro PCB photos, including part numbers, manufacturers, and board context.

---

## ICS 9ZXL1950DKIL

| Field | Value |
|---|---|
| **Manufacturer** | ICS (Integrated Circuit Systems, now Renesas/IDT) |
| **Part Number** | 9ZXL1950DKIL |
| **Function** | Spread-spectrum clock generator / synthesizer |
| **Package** | QFN (observed ~72-pin) |
| **Date Code** | 2403538 |
| **Origin** | TWN (Taiwan), year 2024 (date code 2403538) |
| **PCB** | RBE162548_V52 — RTE162P54B-2UR SFF-8654-8i riser for SXM2 baseboard |
| **Photo** | `PXL_20260322_133229089.MACRO_FOCUS.jpg` |

### Notes
- Used on SFF-8654-8i riser cards to generate reference clocks for PCIe 4.0 / SerDes interfaces.
- The `DKIL` suffix indicates a specific speed/spread variant.
- Identified 2026-03-22 from macro photo of the back side of the RTE162P54B-2UR riser card.

---

## RTE162P54B-2UR — SFF-8654-8i Riser Card for SXM2 Baseboard

| Field | Value |
|---|---|
| **Model** | RTE162P54B-2UR (alias: RBS-16G4-2P54) |
| **Board** | RBE162548_V52 |
| **P/N** | 97.R566V01 |
| **S/N** | YR260130180151 |
| **Function** | SFF-8654-8i riser card — connects SXM2 GPU baseboard via SlimSAS PCIe 4.0 |
| **Connectors** | 2× SFF-8654-8i (SlimSAS x8, internal), bracket labels SLIM6A51 / SLIM6A52 |
| **PCIe bandwidth** | PCIe 4.0 at 16 GT/s per lane; x8 per connector = ~16 GB/s, x16 total = ~32 GB/s |
| **Form factor** | ~147 × 68.5 mm, 2U rack, ~1W, 3.3V PCIe slot powered |
| **Certifications** | CE, FCC, RoHS, UL E511912, 94V-0, date code 0226 |
| **Photos** | `PXL_20260322_133210597.jpg` (back/bracket), `PXL_20260322_133229089.MACRO_FOCUS.jpg` (ICS chip) |

### Notes
- Part of the **P54 connector family** — same SFF-8654-8i SlimSAS connectors as on the TNS-2SXM2-4P54 300G dual-SXM2 baseboard.
- The "SLIM" in SLIM6A51/SLIM6A52 bracket silkscreen refers to SlimSAS (SFF-8654).
- The "P54" suffix in the model number identifies the SFF-8654 (SlimSAS) connector family.
- Primary ASIC (PCIe switch/bridge) on front side not yet photographed — unknown.
- Identified 2026-03-22 from photos of the bracket/back side.

---

## TNS-2SXM2-4P54 — Dual-SXM2 GPU Carrier Baseboard

| Field | Value |
|---|---|
| **Model** | TNS-2SXM2-4P54 |
| **P/N** | 97.R529V07 |
| **S/N** | YR25111801002? (partially legible) |
| **Type** | Dual-SXM2 GPU carrier / baseboard |
| **GPU Slots** | 2× SXM2 (NVIDIA SXM2 form factor; each socket has 2 rectangular connector halves) |
| **Host Interface** | 4× SFF-8654-8i (SlimSAS x8, PCIe 4.0) on left edge, labelled X16/X8 — "4P54" in model name |
| **PCIe bandwidth** | PCIe 4.0, 4× x8 = ~64 GB/s aggregate to host |
| **Power** | 2× large PSU connectors (~20-24 pin) + 2× 8-pin PCIe power + aux connectors on top edge |
| **Companion riser** | RTE162P54B-2UR — 2× per baseboard (each riser carries 2× SFF-8654-8i) |
| **Host platform** | Intel Xeon server (confirmed via `IntelRCSetup` tab in AMI Aptio BIOS v2.18.1263) |
| **GPUs installed** | 2× NVIDIA V100 SXM2 (confirmed by photo `PXL_20260314_180902214.jpg`) |
| **Coolers** | 2× SXM2 heatsink — aluminum fin stack with copper heat pipes |
| **Photos (board)** | `PXL_20260314_104840199.jpg` (front/component side), `PXL_20260314_104823252.jpg` (back/solder side) |
| **Photos (assembled)** | `PXL_20260314_180902214.jpg` (2× V100 + 2× heatsink installed, in shipping box) |
| **Photos (BIOS)** | `PXL_20260322_114203660.MP.jpg` (PCI resource error), `PXL_20260322_125720017.jpg` (CSM config) |

### Physical Description
- Large black PCB, SXM2 sockets arranged vertically (top + bottom), power headers along top edge
- Left edge: 4× SFF-8654-8i stacked in two pairs; silkscreen shows X16 and X8 lane designations per group
- Back/solder side: 2× metal SXM2 socket retainer frames with rectangular cutouts; sparse SMD population
- Right edge: board version/revision silkscreen; CE/FCC marks

### Model Name Breakdown
- **TNS** — product line prefix
- **2SXM2** — 2× SXM2 GPU slots
- **4P54** — 4× SFF-8654 (SlimSAS) ports; same "P54" family identifier as on RTE162P54B-2UR risers

### V100 SXM2 Heatsink Assembly Procedure
Labels printed on each heatsink lid (both heatsinks identical):

| Step | Order |
|---|---|
| Heat sink assembly | A → B → C → D |
| Heat sink disassembly | D → C → B → A |
| GPU screw assembly | 1 → 2 → 3 → 4 → 5 → 6 → 7 → 8 |
| GPU screw disassembly | 8 → 7 → 6 → 5 → 4 → 3 → 2 → 1 |

- Corner screw numbers on heatsink lid: **5** (top-left), **7** (top-right), **8** (bottom-left), **6** (bottom-right)
- A/B/C/D labels indicate heatsink retention bracket screw positions
- 1–8 GPU screws mount the SXM2 module die to the socket; cross-pattern tightening sequence

### Installation Notes
- **PCI Resource Exhaustion (BAR space)**: Populating both SXM2 slots with high-BAR GPUs (e.g., NVIDIA V100 SXM2) alongside other PCIe cards can trigger "Insufficient PCI Resources Detected" in AMI Aptio BIOS.
  - Fix: Enable **Above 4G Decoding** under Advanced → PCI Configuration.
  - On Intel Xeon platforms: also check **IntelRCSetup** → PCIe resource allocation settings.
  - Observed during 2026-03-22 installation session (BIOS v2.18.1263, AMI copyright 2020).
- **CSM**: Host BIOS had CSM enabled with Storage/Video in Legacy mode, Other PCI in UEFI mode. Note that "Above 4G Decoding" typically requires UEFI boot — verify compatibility if CSM is active.

### Kit Contents (from eBay listing 136861481597)
- 1× TNS-2SXM2-4P54 baseboard
- 2× RTE162P54B-2UR risers (PCIe 3.0 x16 → 2× SFF-8654, with full-height and half-height brackets)
- 4× SFF-8654 cables

### Board Features (from community documentation)
- 6× fan headers — supports both DC and PWM fans
- AT/MT power override switch — selects between PCIe-controlled power or always-on
- Onboard power regulation, fan controllers, and thermal/temperature probes
- Debug headers

### Vendor & Sourcing
- Manufacturer: anonymous Chinese OEM ("made by who knows") — no official documentation
- Primary source: eBay item **136861481597**
- Knowledge source: community reverse-engineering and homelab blog posts
- Blog reference: [angrysysadmins.tech — Cheap-ish AI Homelab: V100s, Custom Boards, and NVLink](https://angrysysadmins.tech/index.php/2026/03/grassyloki/cheap-ish-ai-homelab-on-a-budget-v100s-custom-boards-and-nvlink/)

### Compatible GPUs
- **NVIDIA Tesla V100 SXM2** — 16 GB or 32 GB HBM2
- SXM3, SXM4, SXM6 are **not compatible** (different socket/connector)

### NVLink Topology
- **2× V100 SXM2**: Direct NVLink connection — 150 GB/s per direction (half-duplex)
- **4+ GPUs**: Requires NVSwitch chips for full-mesh — not present on this board
- 4-GPU variant with potential NVLink scaling: **YMZX-4GPU-Q1** (different model, eBay item 298083941971)

### BIOS / Platform Requirements
| Requirement | Notes |
|---|---|
| UEFI boot | Legacy/CSM must be **disabled** |
| Above 4G Decoding | Required for high-BAR GPUs like V100 |
| Resizable BAR / Large Bar Support | Required |
| SR-IOV | Recommended |
| PCIe ARI | Recommended |
| Enhanced PCIe error reporting | Recommended where supported |

### Host Platform Compatibility
- **Recommended**: AMD Threadripper (X399 platform) or similar non-proprietary server/workstation with PCIe bifurcation support
- **Dell R740 (iDRAC9) — INCOMPATIBLE**: iDRAC9 fans ramp to 100% ~5 min after boot; thermal controls for SXM2 PCIe slot cannot be suppressed via iDRAC — not usable if noise is a concern
- PCIe bifurcation (x8/x8) required for 2-riser setup; PLX-based riser card available if motherboard lacks native bifurcation
- M.2 M-key to PCIe adapter also viable for Mini PC deployments

### Real-World Performance (Ollama, 2× V100 SXM2 32 GB, AMD Threadripper 2950X)
Tested with ollama 0.18.0, script: [ollama-benchmark](https://github.com/aidatatools/ollama-benchmark)

| Model | Gen Speed (tok/s) |
|---|---|
| qwen3-coder:30b | 98.84 |
| nemotron-3-nano:30b | 122.51 |
| lfm2:24b | 138.48 |
| gpt-oss:20b | 119.43 |
| llama3.2:3b | 207.62 |

### Known Gotchas
- SXM2 modules idle at ~42 W each (poor power saving in manual mode)
- NVIDIA driver support: V100 is supported up to driver branch **580** (CUDA 12) — use `nvidia-580xx-dkms`
- `nvidia-open` kernel drivers do **not** work with V100 — use proprietary branch
- Some AI models/tools require Turing (RTX 20xx) or Ampere (RTX 30xx) — not compatible with V100
- Physical footprint is large; no rack-mount option in the kit
