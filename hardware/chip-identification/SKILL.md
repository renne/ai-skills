---
name: chip-identification
description: Reference for identified chips and ICs from PCB macro photos. Use this skill when identifying or looking up specific ICs/chips found on PCBs. For full card or baseboard documentation, see the dedicated skill for that hardware.
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
| **Photo** | `../rte162p54b-2ur/photos/PXL_20260322_133229089.MACRO_FOCUS.jpg` |

### Notes
- Used on SFF-8654-8i riser cards to generate reference clocks for PCIe 4.0 / SerDes interfaces.
- The `DKIL` suffix indicates a specific speed/spread variant.
- Identified 2026-03-22 from macro photo of the back side of the RTE162P54B-2UR riser card.
- For full riser card documentation, see [hardware/rte162p54b-2ur/SKILL.md](../rte162p54b-2ur/SKILL.md).
