---
name: fujitsu-d3417-b2
description: Hardware and firmware reference for the Fujitsu D3417-B2 desktop/workstation mainboard (Intel Kaby Lake platform). Covers BIOS flash procedures (DOS, Windows, TFTP), key files, and Intel AMT/MEBx management.
---
# Fujitsu D3417-B2 — Hardware Reference

## Overview

The **Fujitsu D3417-B2** is a micro-ATX desktop/workstation mainboard based on the Intel Kaby Lake platform (7th gen Core), used in Fujitsu ESPRIMO and similar systems.

| Attribute | Value |
|-----------|-------|
| Board | Fujitsu D3417-B2 |
| Platform | Intel Kaby Lake (7th gen Core) |
| BIOS vendor | Fujitsu (AMI UEFI base) |
| Source folder | `/home/renne/Dokumente/Geräte/Fujitsu-D3417-B2/` |

---

## BIOS Update

### Version: R1.26.0

| Item | Value |
|------|-------|
| Version string | R1.26.0 |
| Internal ID | V50012R1260 |
| Admin package | `FTS_D3417B2xAdminpackageCompressedFlashFiles_V50012R1260_1231467/` |
| ROM file | `D3417-B2.ROM` (8 MB) |
| Backup file | `D3417-B2x.R1.26.0.bup` |
| Update capsule | `D3417-B2x.R1.26.0.UPC` |
| Source | `/home/renne/Dokumente/Geräte/Fujitsu-D3417-B2/BIOS/FTS_D3417B2xAdminpackageCompressedFlashFiles_V50012R1260_1231467/` |

### Flash Methods

#### DOS / EFI Shell

```
DOS/DosFlash.BAT   — batch file for DOS/EFI environment
DOS/D3417-B2.UPD   — update binary used by DosFlash
DOS/EfiFlash.exe   — EFI shell flash tool
```

**EFI Shell procedure:**
1. Copy `DOS/` folder contents to FAT32 USB stick root.
2. Boot into EFI shell (F12 → EFI Shell).
3. Navigate to USB: `fs0:` (or `fs1:`, etc.)
4. Run `EfiFlash.exe` — it auto-detects the `.UPD` file.

> ⚠️ Unlike D3643-H1, D3417-B2 does **not** use the `EFI/FUJITSU/EfiFlash.efi` pattern. Use `DOS/EfiFlash.exe` instead.

#### Windows

```
Windows/DeskFlash32Bit/           — 32-bit Windows flash utility
Windows/DeskFlash32Bit_UPD.bat    — helper batch
Windows/DeskFlash64Bit/           — 64-bit Windows flash utility
Windows/DeskFlash64Bit_UPD.bat    — helper batch
```

Run `DeskFlash64Bit_UPD.bat` from Windows (admin required).

#### TFTP (Network Flash)

```
TFTP/D3417-B2-1-26-0.UPC   — UEFI update capsule for TFTP delivery
TFTP/D3417-B2.csv           — TFTP configuration/manifest
```

For BMC/iRMC or TFTP-based remote BIOS updates.

---

## Intel AMT / MEBx

The file `meshcmd` at the root of the D3417-B2 folder is the **Intel MEBx management CLI tool** (part of Intel Manageability Commander / MeshCommander):

```
/home/renne/Dokumente/Geräte/Fujitsu-D3417-B2/meshcmd
```

Use to configure or interact with Intel AMT (Active Management Technology) if the D3417-B2 variant includes Intel vPro/AMT support.

Basic usage:
```bash
./meshcmd --help
./meshcmd MEInfo         # Read Management Engine version/status
./meshcmd AmtInfo        # AMT provisioning info
```

---

## References

- [Fujitsu D3417-B2 BIOS Downloads (Fujitsu Support)](https://support.ts.fujitsu.com/IndexDownload.asp?lng=de&type=10&SoftwareGuid=)
- [Intel AMT / MeshCommander](https://www.meshcommander.com/)
