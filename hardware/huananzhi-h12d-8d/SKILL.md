---
name: huananzhi-h12d-8d
description: Hardware reference for the Huananzhi H12D-8D dual-socket AMD EPYC server mainboard. Use this skill when asked about memory slot numbering, DIMM population order, board layout, supported CPUs, or other hardware specifications for this board.
---
# Huananzhi H12D-8D Hardware Reference

## Overview

The **Huananzhi H12D-8D** (华南金牌 H12D-8D) is a dual-socket AMD EPYC server mainboard.

- **Form factor:** Extended ATX (server)
- **CPU sockets:** 2× SP3 (AMD EPYC)
- **Supported CPUs:** AMD EPYC 7001 (Naples), 7002 (Rome), 7H12 series
- **Memory type:** DDR4 ECC RDIMM / LRDIMM
- **Memory slots:** 8 total (4 per CPU)
- **Memory configuration:** Octa-channel (8-channel) across both CPUs

## Memory Slot Numbering

Slots are **labeled MM1–MM8** on the board silkscreen.

| Slot | CPU  | Channel | Notes                         |
|------|------|---------|-------------------------------|
| MM1  | CPU1 | A       | **First slot — install here first** |
| MM2  | CPU1 | B       |                               |
| MM3  | CPU1 | C       |                               |
| MM4  | CPU1 | D       |                               |
| MM5  | CPU2 | A       | First slot for second CPU     |
| MM6  | CPU2 | B       |                               |
| MM7  | CPU2 | C       |                               |
| MM8  | CPU2 | D       |                               |

**MM1 is the first slot**, physically located closest to CPU socket 1 (CPU1).

## DIMM Population Rules

| Configuration        | Install in                    |
|----------------------|-------------------------------|
| 1 DIMM               | MM1                           |
| 2 DIMMs (one CPU)    | MM1, MM3 (channels A + C)     |
| 2 DIMMs (dual CPU)   | MM1 + MM5 (one per CPU, max bandwidth) |
| 4 DIMMs              | MM1–MM4                       |
| 8 DIMMs (full)       | MM1–MM8                       |

- Always populate in matched pairs per CPU for multi-channel bandwidth.
- For a single installed CPU (CPU1 only), install DIMMs in MM1–MM4.

## Known Quirks and Limitations

- ⚠️ The official Huananzhi PDF manual (`http://www.huananzhi.com/uploads/file/20260212/1770877734213650.pdf`) is slow to load and frequently times out. Use the Manualzz mirror instead.
- The board may require BIOS update for EPYC 7002 (Rome) CPUs even though it is listed as supported — check the Huananzhi product page for BIOS versions.
- Some online sources list this board as "H12D-8D" while the physical silk screen shows "H12D-8D". Both refer to the same product.

## References

- [User Manual on Manualzz](https://manualzz.com/doc/84402668/huananzhi-h12d-8d-user-manual) — English mirror of official manual
- [Official Huananzhi product page (EN)](http://www.huananzhi.com/en/list_6/183.html)
- [Official manual PDF (CN)](http://www.huananzhi.com/uploads/file/20260212/1770877734213650.pdf) — may time out
