---
name: huananzhi-h12d-8d
description: Hardware reference for the Huananzhi H12D-8D single-socket AMD EPYC server mainboard (Ver2.0). Use this skill when asked about memory slot numbering, DIMM population order, board layout, supported CPUs, expansion slots, rear I/O, internal headers, BIOS/IPMI, or any other hardware specifications for this board.
---
# Huananzhi H12D-8D Hardware Reference

## Overview

The **Huananzhi H12D-8D** (华南金牌 H12D-8D, Ver2.0) is a **single-socket** AMD EPYC ATX server mainboard released in early 2025. It targets high-performance computing, data center, workstation, and AI workloads.

> ⚠️ Despite the "D" in H12**D**, this board has only **one** CPU socket (not dual-socket).

| Attribute           | Value                                      |
|---------------------|--------------------------------------------|
| Form factor         | ATX, 305 × 245 mm                          |
| PCB layers          | 14-layer                                   |
| CPU socket          | 1× AMD SP3                                 |
| Supported CPUs      | EPYC 7002 (Rome, Zen2), EPYC 7003 (Milan, Zen3) |
| **Not** supported   | EPYC 7001 (Naples), EPYC 9004 (Genoa, SP5), EPYC 8004 (Siena, SP6) |
| Memory slots        | 8× DDR4 ECC RDIMM / LRDIMM                |
| Memory channels     | 8-channel (octa-channel)                   |
| Memory speeds       | 2133 / 2400 / 2666 / 2933 / 3200 MT/s     |
| Max RAM             | Up to 2 TB                                 |
| BIOS                | AMI UEFI                                   |
| OS support          | Windows, Linux                             |
| Approx. price (CN)  | ¥2188 (no BMC) / ¥2388 (with BMC module)   |

## Memory Slot Numbering

Slots are **labeled MM1–MM8** on the board silkscreen. All 8 slots serve the single CPU.

| Slot | Channel | Notes                               |
|------|---------|-------------------------------------|
| MM1  | A       | **First slot — install here first** |
| MM2  | B       |                                     |
| MM3  | C       |                                     |
| MM4  | D       |                                     |
| MM5  | E       |                                     |
| MM6  | F       |                                     |
| MM7  | G       |                                     |
| MM8  | H       |                                     |

**MM1 is the first slot** on this board.

## DIMM Population Rules

Spread DIMMs across channels before doubling up in any channel.

| DIMMs installed | Recommended slots         |
|-----------------|---------------------------|
| 1               | MM1                       |
| 2               | MM1, MM5                  |
| 4               | MM1, MM3, MM5, MM7        |
| 8 (full/optimal)| MM1–MM8                   |

- Use identical DIMMs (capacity, speed, rank) for best compatibility.
- 8 DIMMs gives maximum bandwidth via true 8-channel interleaving.

## Expansion Slots & Storage

### PCIe & M.2
- 4× PCIe 4.0 x16 slots
- 3× M.2 2280 NVMe (PCIe 4.0 x4 each)

### Onboard SATA
- 4× SATA3 (6 Gbps) ports directly on the board

### SFF-8643 (Mini-SAS HD) — Dual-Mode
- 3× SFF-8643 ports — each port operates in **one** of two modes (configure via BIOS):
  - **SATA mode:** each SFF-8643 breaks out into 4× SATA3 ports → up to **12 additional SATA ports** total
  - **U.2 mode:** each port provides 1× PCIe 4.0 x4 NVMe → up to **3× U.2 NVMe drives** total
  - ⚠️ Cannot mix SATA and U.2 modes simultaneously on the same SFF-8643 port.

## Power

| Connector        | Details                                 |
|------------------|-----------------------------------------|
| 24-pin ATX       | Main board power                        |
| 2× 8-pin EPS     | CPU power (both required under load)    |
| VRM              | 9+5 phase, 60 A DrMOS per phase         |

## Networking

| Interface              | Chip                  | Notes                              |
|------------------------|-----------------------|------------------------------------|
| 2× RJ45 (2.5 GbE)      | Intel I226-V          | Main data network                  |
| 1× RJ45 (100 MbE)      | Realtek RTL8201F      | Dedicated IPMI/BMC management port |

## Rear I/O

- 4× USB 3.2 Gen1 Type-A
- 2× USB 2.0 Type-A
- 2× RJ45 (Intel I226-V 2.5 GbE)
- 1× RJ45 (IPMI/BMC management, Realtek RTL8201F)
- 1× VGA (only active when AST2500 BMC module is installed)
- 1× 2-hole audio jack (USB digital audio via CJC6811A SoC)
- 1× I/O shield (included)

## Internal Headers

| Header            | Connector         | Function                              |
|-------------------|-------------------|---------------------------------------|
| USB 3.2 Gen1      | 19-pin            | Front panel USB 3.x                   |
| USB 2.0           | 9-pin             | Front panel USB 2.0                   |
| Audio             | Standard          | Front panel audio                     |
| COM               | Serial header     | COM port                              |
| SMBUS             | —                 | System Management Bus                 |
| NCSI              | —                 | Network Controller Sideband Interface |
| TPM2.0            | —                 | Hardware TPM module                   |
| PMBus             | —                 | Power Management Bus                  |
| MB BIOS           | —                 | BIOS flash/recovery header            |
| DE_BUG            | —                 | Onboard POST code debug display       |
| System Restart    | —                 | System restart header                 |
| F_Panel           | Standard          | Power SW, Reset SW, Power LED, HDD LED|

## Cooling & Fan Headers

- 6× fan headers total:
  - 1× **CPU_FAN1** — PWM, BIOS-controlled
  - 5× **SYS_FAN1–SYS_FAN5** — PWM, can be BMC-controlled when BMC module installed
- Fan curves configurable via AMI UEFI BIOS; SYS fans can be managed remotely via IPMI (requires BMC module).

## Audio

- **SoC:** Guanghua Xin (光华芯) **CJC6811A** USB digital audio codec
- Connected to rear 2-hole audio jack and internal audio header

## BMC / IPMI Management

- **BMC module:** ASPEED **AST2500** — sold **separately** (not included by default)
- Installing the BMC module enables:
  - IPMI 2.0 remote management
  - VGA output on rear I/O
  - Remote KVM, power control, sensor monitoring
  - BMC-based SYS_FAN management
- IPMI 2.0 compliant when BMC is installed

## BIOS

- **Type:** AMI UEFI
- **CPU support:** EPYC 7002 (Rome) native; EPYC 7003 (Milan) may require BIOS update — check Huananzhi product page
- **Configuration options include:** CPU cores, virtualization (SVM), memory channels/speeds, fan curves, NVMe/SATA boot priority, PCIe mode for SFF-8643, IPMI settings
- **BIOS update tutorial:** https://www.youtube.com/watch?v=nCq0WRROdS8
- **BIOS download:** http://www.huananzhi.com/en/list_6/183.html

## Package Contents

- 1× H12D-8D motherboard
- 2× SATA cables
- 1× I/O shield
- 1× User manual
- *(BMC/IPMI module sold separately)*

## Known Quirks and Limitations

- ⚠️ **Single-socket:** Despite the "D" in H12**D**, only one SP3 CPU socket is present.
- ⚠️ **No Naples support:** EPYC 7001 (Naples) is NOT supported — only Rome (7002) and Milan (7003).
- ⚠️ **SFF-8643 mode is exclusive:** One mode (SATA or U.2) per port — cannot mix.
- ⚠️ **VGA requires BMC:** Rear VGA output is only functional when the optional AST2500 BMC module is installed.
- ⚠️ **No EPYC 9004/8004 support:** These CPUs use SP5/SP6 sockets, incompatible with SP3.
- 📝 **Ver2.0 designation** appears on official product page — may indicate a board revision.
- 📝 Some Sohu articles claim "EPYC 8000 series" compatibility — these appear to be clickbait/AI-generated titles; all verified sources confirm only 7002/7003 support.
- 📝 Manual PDF on Huananzhi site frequently times out; use the Manualzz mirror.
- 📝 ES (engineering sample) CPU compatibility is inconsistent and depends on AGESA/microcode version in the installed BIOS.

## References

- [User Manual on Manualzz](https://manualzz.com/doc/84402668/huananzhi-h12d-8d-user-manual) — English mirror (most reliable)
- [Official Huananzhi product page (EN)](http://www.huananzhi.com/en/list_6/183.html) — BIOS & driver downloads
- [Official manual PDF (CN)](http://www.huananzhi.com/uploads/file/20260212/1770877734213650.pdf) — may time out
- [ServeTheHome forum thread — ES CPU discussion](https://forums.servethehome.com/index.php?threads/epyc-es-series-on-huananzhi-h12d-8d.50421/)
- [BIOS Update Tutorial on YouTube](https://www.youtube.com/watch?v=nCq0WRROdS8)
- [AliExpress listing with specs](https://www.aliexpress.com/item/1005008701050951.html)
