---
name: NVIDIA GeForce GT 710 (GK208B)
description: NVIDIA GeForce GT 710 — GK208B GPU, PCIe 2.0 x8/x16, 1–2 GB DDR3/GDDR5 — VBIOS, hardware identifiers, flashing notes
---

# NVIDIA GeForce GT 710 (GK208B)

## Overview

| Field | Value |
|-------|-------|
| Model | GeForce GT 710 |
| GPU | GK208B (Kepler architecture) |
| PCI Vendor ID | `0x10DE` (NVIDIA) |
| PCI Device ID | `0x128B` |
| Board | P2132 SKU 14 |
| Interface | PCIe 2.0 x8 (x16 slot compatible) |
| VRAM | 1 GB or 2 GB DDR3 or GDDR5 (varies by SKU) |
| TDP | ~19 W (passively cooled on most SKUs) |
| Display outputs | Varies (HDMI / DVI / VGA common) |
| Driver support | NVIDIA legacy driver branch 470.xx (Linux) |

## VBIOS

### File: `vbios/190890.rom`

| Field | Value |
|-------|-------|
| Source | [TechPowerUp VBIOS Database #190890](https://www.techpowerup.com/vgabios/190890/190890) |
| Version | 80.28.A6.00.10 |
| Board string | `GK208 Board - 21320014` |
| PCI Vendor ID | `0x10DE` |
| PCI Device ID | `0x128B` |
| Board SKU | P2132 SKU 14 |
| File size | 169472 bytes (166 KB) |
| MD5 | `4565d08a37b448d6c869e7b834dc04a4` |

### Verify identifiers before flashing

```bash
# Check PCI IDs embedded in the ROM
python3 -c "
import struct
data = open('vbios/190890.rom','rb').read()
idx = data.find(b'PCIR')
vendor = struct.unpack_from('<H', data, idx+4)[0]
device = struct.unpack_from('<H', data, idx+6)[0]
print(f'VendorID=0x{vendor:04x} DeviceID=0x{device:04x}')
"

# Match against installed card
lspci -nn | grep -i nvidia
# Expect: [10de:128b]
```

### Flashing

```bash
# Linux — nvflash (NVIDIA card must be present and accessible)
sudo nvflash --protectoff
sudo nvflash -6 vbios/190890.rom

# Or with confirmation bypass:
sudo nvflash --index=0 -6 vbios/190890.rom
```

> **Warning:** Only flash if PCI Device ID `0x128B` and board `P2132 SKU 14` match your card exactly.
> A wrong VBIOS can brick the card. Always verify MD5 of the ROM file first.

## Driver Support

| OS | Driver |
|----|--------|
| Windows 10/11 | GeForce Game Ready 474.x (legacy) |
| Linux | `nvidia` 470.xx branch (last supporting Kepler) |
| Linux | `nouveau` open-source (limited 3D) |

```bash
# Install legacy driver on Debian/Ubuntu
sudo apt install nvidia-legacy-470xx-driver
```

## Notes

- GT 710 is a **passive/low-profile** card popular as a secondary display or KVM GPU alongside compute cards
- On systems with multiple GPUs, useful as the CSM/legacy display adapter so primary compute GPUs can have UEFI GOP
- Kepler architecture (GK208) is **not** supported by the `nvidia-open` kernel module — must use closed-source driver
- PCIe power connector: **none required** (powered entirely from PCIe slot)
